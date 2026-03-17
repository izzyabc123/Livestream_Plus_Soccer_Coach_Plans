---
name: YouTube Livestream - 100 User Scale
overview: Intermediate mobile app (iOS + Android) for ~100 users with YouTube API integration, allowing in-app broadcasting and viewing. Bridges gap between MVP and full production.
todos:
  - id: simple-backend
    content: Set up lightweight backend (Node.js/Express with PostgreSQL)
    status: pending
  - id: youtube-oauth
    content: Implement YouTube OAuth 2.0 flow for both platforms
    status: pending
  - id: youtube-api-integration
    content: Integrate YouTube Data API v3 and Live Streaming API
    status: pending
  - id: android-enhanced
    content: Enhance Android app with YouTube API integration
    status: pending
  - id: ios-app
    content: Build iOS app with YouTube API integration
    status: pending
  - id: basic-realtime
    content: Add basic real-time updates (polling or simple WebSocket)
    status: pending
  - id: basic-monitoring
    content: Set up basic monitoring and error tracking
    status: pending
  - id: deployment
    content: Deploy to single cloud server (DigitalOcean/Railway)
    status: pending
isProject: true
---

# YouTube Livestream - 100 User Scale

## High-Level Architecture

```mermaid
graph TB
    subgraph "Client Layer"
        iOS[iOS App Swift]
        Android[Android App Kotlin]
    end
    
    subgraph "Single Server VPS"
        API[Node.js API<br/>Express + PostgreSQL]
    end
    
    subgraph "External Services"
        YouTube[YouTube API]
        Firebase[Firebase FCM]
    end
    
    iOS --> API
    Android --> API
    
    API --> YouTube
    API --> Firebase
```



## System Architecture Comparison

### MVP vs 100-User vs Full Production


| Component               | MVP (5-10 users)      | 100-User Scale            | Full Production (1000s+) |
| ----------------------- | --------------------- | ------------------------- | ------------------------ |
| **Backend**             | None (Firebase only)  | Single Node.js server     | Load-balanced cluster    |
| **Database**            | Firestore             | PostgreSQL on same server | PostgreSQL + replicas    |
| **Cache**               | None                  | Optional (in-memory)      | Redis cluster            |
| **Auth**                | Firebase Auth         | JWT + YouTube OAuth       | Same + advanced security |
| **YouTube Integration** | Intent (launches app) | YouTube API (in-app)      | Same                     |
| **iOS Support**         | None                  | Native Swift app          | Same                     |
| **Real-time Updates**   | Firestore listeners   | Polling (simple)          | WebSocket cluster        |
| **Hosting**             | Free (Firebase)       | Single $12-24/month VPS   | $200-2000/month          |
| **Monitoring**          | None                  | Basic logging + Sentry    | Full observability stack |
| **CDN**                 | None                  | None (not needed)         | CloudFront/Cloudflare    |
| **Deployment**          | Firebase CLI          | Single docker-compose     | Kubernetes/ECS           |
| **Cost**                | $0/month              | ~$25-50/month             | ~$200-2000/month         |


## Key Philosophy for 100-User Scale

**Keep it simple, but structured**:

✅ Single VPS server (not serverless, not multi-server cluster)
✅ PostgreSQL on same machine (no managed database service)
✅ Lightweight monitoring (not full observability stack)
✅ Basic polling for updates (not complex WebSocket infrastructure)
✅ Simple deployment (docker-compose, not Kubernetes)
✅ Focus on features over infrastructure
✅ Easy to understand and maintain by 1-2 developers

## Core Infrastructure

### 1. Backend API Server

**Technology Stack**:

- **Runtime**: Node.js 20 LTS
- **Framework**: Express.js
- **Database**: PostgreSQL 16
- **ORM**: Prisma or Drizzle
- **Auth**: JWT + Passport.js
- **YouTube**: googleapis npm package

**Simple Server Structure**:

```
backend/
├── src/
│   ├── routes/
│   │   ├── auth.js          # Login, register, YouTube OAuth
│   │   ├── streams.js       # Stream management
│   │   └── users.js         # User profile
│   ├── services/
│   │   ├── youtube.js       # YouTube API calls
│   │   └── auth.js          # JWT handling
│   ├── middleware/
│   │   ├── auth.js          # JWT verification
│   │   └── error.js         # Error handling
│   ├── models/
│   │   ├── User.js
│   │   └── Stream.js
│   ├── db.js                # Database connection
│   └── server.js            # Main entry point
├── package.json
├── .env
└── Dockerfile
```

**Essential API Endpoints**:

```typescript
// Authentication
POST   /api/auth/register
POST   /api/auth/login
GET    /api/auth/youtube/authorize      # Start OAuth flow
GET    /api/auth/youtube/callback       # OAuth callback
POST   /api/auth/refresh

// Streams
GET    /api/streams                     # List live streams
GET    /api/streams/:id
POST   /api/streams                     # Create YouTube broadcast
POST   /api/streams/:id/start
POST   /api/streams/:id/stop
DELETE /api/streams/:id

// Users
GET    /api/users/me
PUT    /api/users/me
GET    /api/users/:id/streams
```

**package.json**:

```json
{
  "name": "livestream-backend",
  "version": "1.0.0",
  "dependencies": {
    "express": "^4.18.0",
    "pg": "^8.11.0",
    "prisma": "^5.9.0",
    "@prisma/client": "^5.9.0",
    "jsonwebtoken": "^9.0.0",
    "bcrypt": "^5.1.0",
    "googleapis": "^131.0.0",
    "dotenv": "^16.4.0",
    "cors": "^2.8.5",
    "helmet": "^7.1.0",
    "express-rate-limit": "^7.1.0",
    "@sentry/node": "^7.100.0"
  },
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "db:migrate": "prisma migrate dev",
    "db:generate": "prisma generate"
  }
}
```

### 2. Database Schema (PostgreSQL)

**Simplified Schema for 100 Users**:

```sql
-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email VARCHAR(255) UNIQUE NOT NULL,
    username VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    avatar_url TEXT,
    youtube_channel_id VARCHAR(100),
    youtube_refresh_token TEXT, -- Encrypted
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_username ON users(username);

-- Streams table
CREATE TABLE streams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    youtube_broadcast_id VARCHAR(100) UNIQUE,
    youtube_video_id VARCHAR(100),
    youtube_stream_key TEXT, -- Encrypted
    rtmp_url TEXT,
    status VARCHAR(50) DEFAULT 'created', -- created, live, ended
    is_public BOOLEAN DEFAULT TRUE,
    started_at TIMESTAMP,
    ended_at TIMESTAMP,
    viewer_count INTEGER DEFAULT 0,
    thumbnail_url TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_streams_user_id ON streams(user_id);
CREATE INDEX idx_streams_status ON streams(status);
CREATE INDEX idx_streams_live ON streams(status) WHERE status = 'live';

-- Simple analytics (optional, for basic tracking)
CREATE TABLE stream_views (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL REFERENCES streams(id) ON DELETE CASCADE,
    user_id UUID REFERENCES users(id) ON DELETE SET NULL,
    viewed_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_views_stream ON stream_views(stream_id);
```

**Prisma Schema** (alternative to raw SQL):

```prisma
// schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
}

model User {
  id                    String   @id @default(uuid())
  email                 String   @unique
  username              String   @unique
  passwordHash          String?  @map("password_hash")
  avatarUrl             String?  @map("avatar_url")
  youtubeChannelId      String?  @map("youtube_channel_id")
  youtubeRefreshToken   String?  @map("youtube_refresh_token")
  createdAt             DateTime @default(now()) @map("created_at")
  updatedAt             DateTime @updatedAt @map("updated_at")
  
  streams               Stream[]
  views                 StreamView[]
  
  @@map("users")
}

model Stream {
  id                 String   @id @default(uuid())
  userId             String   @map("user_id")
  title              String
  description        String?
  youtubeBroadcastId String?  @unique @map("youtube_broadcast_id")
  youtubeVideoId     String?  @map("youtube_video_id")
  youtubeStreamKey   String?  @map("youtube_stream_key")
  rtmpUrl            String?  @map("rtmp_url")
  status             String   @default("created")
  isPublic           Boolean  @default(true) @map("is_public")
  startedAt          DateTime? @map("started_at")
  endedAt            DateTime? @map("ended_at")
  viewerCount        Int      @default(0) @map("viewer_count")
  thumbnailUrl       String?  @map("thumbnail_url")
  createdAt          DateTime @default(now()) @map("created_at")
  updatedAt          DateTime @updatedAt @map("updated_at")
  
  user               User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  views              StreamView[]
  
  @@index([userId])
  @@index([status])
  @@map("streams")
}

model StreamView {
  id        String   @id @default(uuid())
  streamId  String   @map("stream_id")
  userId    String?  @map("user_id")
  viewedAt  DateTime @default(now()) @map("viewed_at")
  
  stream    Stream   @relation(fields: [streamId], references: [id], onDelete: Cascade)
  user      User?    @relation(fields: [userId], references: [id], onDelete: SetNull)
  
  @@index([streamId])
  @@map("stream_views")
}
```

### 3. YouTube API Integration

**OAuth 2.0 Flow** (simplified for single server):

```javascript
// services/youtube.js
const { google } = require('googleapis');
const youtube = google.youtube('v3');

class YouTubeService {
  constructor() {
    this.oauth2Client = new google.auth.OAuth2(
      process.env.YOUTUBE_CLIENT_ID,
      process.env.YOUTUBE_CLIENT_SECRET,
      process.env.YOUTUBE_REDIRECT_URI
    );
  }

  // Generate OAuth URL
  getAuthUrl() {
    return this.oauth2Client.generateAuthUrl({
      access_type: 'offline',
      scope: [
        'https://www.googleapis.com/auth/youtube',
        'https://www.googleapis.com/auth/youtube.force-ssl',
      ],
      prompt: 'consent', // Force to get refresh token
    });
  }

  // Exchange code for tokens
  async getTokens(code) {
    const { tokens } = await this.oauth2Client.getToken(code);
    return tokens;
  }

  // Create broadcast
  async createBroadcast(userId, { title, description, scheduledStartTime }) {
    const user = await db.users.findById(userId);
    this.oauth2Client.setCredentials({
      refresh_token: decrypt(user.youtubeRefreshToken),
    });

    // Create broadcast
    const broadcastResponse = await youtube.liveBroadcasts.insert({
      auth: this.oauth2Client,
      part: ['snippet', 'contentDetails', 'status'],
      requestBody: {
        snippet: {
          title,
          description,
          scheduledStartTime: scheduledStartTime || new Date().toISOString(),
        },
        contentDetails: {
          enableAutoStart: false,
          enableAutoStop: false,
        },
        status: {
          privacyStatus: 'public',
        },
      },
    });

    // Create stream
    const streamResponse = await youtube.liveStreams.insert({
      auth: this.oauth2Client,
      part: ['snippet', 'cdn'],
      requestBody: {
        snippet: {
          title: `${title} - Stream`,
        },
        cdn: {
          frameRate: '30fps',
          ingestionType: 'rtmp',
          resolution: '1080p',
        },
      },
    });

    // Bind stream to broadcast
    await youtube.liveBroadcasts.bind({
      auth: this.oauth2Client,
      id: broadcastResponse.data.id,
      part: ['id'],
      streamId: streamResponse.data.id,
    });

    return {
      broadcastId: broadcastResponse.data.id,
      videoId: broadcastResponse.data.id,
      streamId: streamResponse.data.id,
      streamKey: streamResponse.data.cdn.ingestionInfo.streamName,
      rtmpUrl: streamResponse.data.cdn.ingestionInfo.ingestionAddress,
    };
  }

  // Transition broadcast (testing -> live -> complete)
  async transitionBroadcast(userId, broadcastId, status) {
    const user = await db.users.findById(userId);
    this.oauth2Client.setCredentials({
      refresh_token: decrypt(user.youtubeRefreshToken),
    });

    return await youtube.liveBroadcasts.transition({
      auth: this.oauth2Client,
      broadcastStatus: status, // 'testing', 'live', or 'complete'
      id: broadcastId,
      part: ['status'],
    });
  }

  // Get broadcast status
  async getBroadcastStatus(userId, broadcastId) {
    const user = await db.users.findById(userId);
    this.oauth2Client.setCredentials({
      refresh_token: decrypt(user.youtubeRefreshToken),
    });

    const response = await youtube.liveBroadcasts.list({
      auth: this.oauth2Client,
      part: ['status', 'contentDetails'],
      id: [broadcastId],
    });

    return response.data.items[0];
  }
}

module.exports = new YouTubeService();
```

**Routes Implementation**:

```javascript
// routes/streams.js
const express = require('express');
const router = express.Router();
const youtubeService = require('../services/youtube');
const { requireAuth } = require('../middleware/auth');

// List live streams (public)
router.get('/', async (req, res) => {
  const streams = await db.streams.findMany({
    where: { status: 'live', isPublic: true },
    include: { user: { select: { id: true, username: true, avatarUrl: true } } },
    orderBy: { startedAt: 'desc' },
  });
  res.json(streams);
});

// Create broadcast
router.post('/', requireAuth, async (req, res) => {
  const { title, description } = req.body;
  
  // Create YouTube broadcast via API
  const ytBroadcast = await youtubeService.createBroadcast(req.user.id, {
    title,
    description,
  });

  // Save to database
  const stream = await db.streams.create({
    data: {
      userId: req.user.id,
      title,
      description,
      youtubeBroadcastId: ytBroadcast.broadcastId,
      youtubeVideoId: ytBroadcast.videoId,
      youtubeStreamKey: encrypt(ytBroadcast.streamKey),
      rtmpUrl: ytBroadcast.rtmpUrl,
      status: 'created',
    },
  });

  res.json({
    stream,
    rtmpUrl: ytBroadcast.rtmpUrl,
    streamKey: ytBroadcast.streamKey,
  });
});

// Start streaming
router.post('/:id/start', requireAuth, async (req, res) => {
  const stream = await db.streams.findById(req.params.id);
  
  if (stream.userId !== req.user.id) {
    return res.status(403).json({ error: 'Unauthorized' });
  }

  // Transition YouTube broadcast to 'live'
  await youtubeService.transitionBroadcast(
    req.user.id,
    stream.youtubeBroadcastId,
    'live'
  );

  // Update database
  await db.streams.update({
    where: { id: req.params.id },
    data: { status: 'live', startedAt: new Date() },
  });

  res.json({ success: true });
});

// Stop streaming
router.post('/:id/stop', requireAuth, async (req, res) => {
  const stream = await db.streams.findById(req.params.id);
  
  if (stream.userId !== req.user.id) {
    return res.status(403).json({ error: 'Unauthorized' });
  }

  // Transition YouTube broadcast to 'complete'
  await youtubeService.transitionBroadcast(
    req.user.id,
    stream.youtubeBroadcastId,
    'complete'
  );

  // Update database
  await db.streams.update({
    where: { id: req.params.id },
    data: { status: 'ended', endedAt: new Date() },
  });

  res.json({ success: true });
});

module.exports = router;
```

### 4. iOS App (Swift)

**Simplified Project Structure**:

```
iOS/
├── LivestreamApp/
│   ├── App/
│   │   └── LivestreamApp.swift
│   ├── Core/
│   │   ├── API/
│   │   │   ├── APIClient.swift
│   │   │   └── APIModels.swift
│   │   └── Auth/
│   │       └── AuthManager.swift
│   ├── Features/
│   │   ├── Auth/
│   │   │   └── LoginView.swift
│   │   ├── StreamList/
│   │   │   ├── StreamListView.swift
│   │   │   └── StreamCardView.swift
│   │   ├── Broadcast/
│   │   │   ├── BroadcastSetupView.swift
│   │   │   └── BroadcastView.swift
│   │   └── Player/
│   │       └── PlayerView.swift
│   └── Models/
│       ├── User.swift
│       └── Stream.swift
└── Package.swift
```

**Key Dependencies**:

```swift
// Package.swift
dependencies: [
    .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0"),
    .package(url: "https://github.com/youtube/youtube-ios-player-helper", from: "1.0.4"),
    .package(url: "https://github.com/HaishinKit/HaishinKit.swift", from: "1.6.0"), // RTMP
]
```

**API Client**:

```swift
// Core/API/APIClient.swift
import Foundation
import Alamofire

class APIClient {
    static let shared = APIClient()
    private let baseURL = "https://your-server.com/api"
    
    private var headers: HTTPHeaders {
        var headers = HTTPHeaders()
        if let token = AuthManager.shared.token {
            headers.add(.authorization(bearerToken: token))
        }
        return headers
    }
    
    // List streams
    func getStreams() async throws -> [Stream] {
        let response = try await AF.request(
            "\(baseURL)/streams",
            headers: headers
        ).serializingDecodable([Stream].self).value
        
        return response
    }
    
    // Create broadcast
    func createBroadcast(title: String, description: String) async throws -> BroadcastResponse {
        let params: [String: Any] = [
            "title": title,
            "description": description
        ]
        
        let response = try await AF.request(
            "\(baseURL)/streams",
            method: .post,
            parameters: params,
            encoding: JSONEncoding.default,
            headers: headers
        ).serializingDecodable(BroadcastResponse.self).value
        
        return response
    }
    
    // Start stream
    func startStream(id: String) async throws {
        try await AF.request(
            "\(baseURL)/streams/\(id)/start",
            method: .post,
            headers: headers
        ).serializingData().value
    }
    
    // Stop stream
    func stopStream(id: String) async throws {
        try await AF.request(
            "\(baseURL)/streams/\(id)/stop",
            method: .post,
            headers: headers
        ).serializingData().value
    }
}

struct BroadcastResponse: Codable {
    let stream: Stream
    let rtmpUrl: String
    let streamKey: String
}
```

**Broadcasting View**:

```swift
// Features/Broadcast/BroadcastView.swift
import SwiftUI
import HaishinKit
import AVFoundation

struct BroadcastView: View {
    @StateObject private var viewModel = BroadcastViewModel()
    @State private var title = ""
    @State private var description = ""
    @State private var isStreaming = false
    
    var body: some View {
        ZStack {
            // Camera preview
            RTMPCameraView(viewModel: viewModel)
                .edgesIgnoringSafeArea(.all)
            
            VStack {
                Spacer()
                
                if !isStreaming {
                    // Setup controls
                    VStack(spacing: 16) {
                        TextField("Stream Title", text: $title)
                            .textFieldStyle(RoundedBorderTextFieldStyle())
                        
                        TextField("Description", text: $description)
                            .textFieldStyle(RoundedBorderTextFieldStyle())
                        
                        Button("Start Broadcast") {
                            Task {
                                await viewModel.startBroadcast(
                                    title: title,
                                    description: description
                                )
                                isStreaming = true
                            }
                        }
                        .buttonStyle(.borderedProminent)
                    }
                    .padding()
                    .background(.ultraThinMaterial)
                } else {
                    // Live controls
                    HStack {
                        Text("🔴 LIVE")
                            .foregroundColor(.red)
                            .fontWeight(.bold)
                        
                        Spacer()
                        
                        Text("Viewers: \(viewModel.viewerCount)")
                        
                        Spacer()
                        
                        Button("Stop") {
                            Task {
                                await viewModel.stopBroadcast()
                                isStreaming = false
                            }
                        }
                        .buttonStyle(.bordered)
                    }
                    .padding()
                    .background(.ultraThinMaterial)
                }
            }
        }
    }
}

class BroadcastViewModel: ObservableObject {
    @Published var viewerCount = 0
    @Published var isStreaming = false
    
    private var rtmpConnection = RTMPConnection()
    private var rtmpStream: RTMPStream!
    private var streamId: String?
    
    init() {
        rtmpStream = RTMPStream(connection: rtmpConnection)
        setupStream()
    }
    
    private func setupStream() {
        rtmpStream.videoSettings = [
            .width: 1280,
            .height: 720,
            .bitrate: 2500 * 1000,
            .profileLevel: kVTProfileLevel_H264_Main_AutoLevel,
        ]
        
        rtmpStream.audioSettings = [
            .bitrate: 128 * 1000,
        ]
        
        rtmpStream.attachCamera(DeviceUtil.device(withPosition: .back))
        rtmpStream.attachAudio(AVCaptureDevice.default(for: .audio))
    }
    
    func startBroadcast(title: String, description: String) async {
        do {
            // Create broadcast via API
            let response = try await APIClient.shared.createBroadcast(
                title: title,
                description: description
            )
            
            streamId = response.stream.id
            
            // Connect to RTMP
            rtmpConnection.connect("\(response.rtmpUrl)")
            rtmpStream.publish(response.streamKey)
            
            // Notify backend to transition to 'live'
            try await APIClient.shared.startStream(id: response.stream.id)
            
            isStreaming = true
            
            // Start polling viewer count
            startViewerCountPolling()
        } catch {
            print("Failed to start broadcast: \(error)")
        }
    }
    
    func stopBroadcast() async {
        guard let streamId = streamId else { return }
        
        rtmpStream.close()
        rtmpConnection.close()
        
        try? await APIClient.shared.stopStream(id: streamId)
        
        isStreaming = false
    }
    
    private func startViewerCountPolling() {
        // Simple polling every 10 seconds
        Timer.scheduledTimer(withTimeInterval: 10.0, repeats: true) { [weak self] _ in
            guard let self = self, self.isStreaming else { return }
            // Fetch viewer count from backend
            Task {
                // Implementation depends on your backend endpoint
            }
        }
    }
}
```

**Stream Player View**:

```swift
// Features/Player/PlayerView.swift
import SwiftUI
import YouTubeiOSPlayerHelper

struct PlayerView: View {
    let stream: Stream
    
    var body: some View {
        VStack(alignment: .leading) {
            YouTubePlayer(videoId: stream.youtubeVideoId ?? "")
                .frame(height: 250)
            
            VStack(alignment: .leading, spacing: 8) {
                Text(stream.title)
                    .font(.headline)
                
                HStack {
                    AsyncImage(url: URL(string: stream.user.avatarUrl ?? "")) { image in
                        image.resizable()
                    } placeholder: {
                        Color.gray
                    }
                    .frame(width: 32, height: 32)
                    .clipShape(Circle())
                    
                    Text(stream.user.username)
                        .font(.subheadline)
                    
                    Spacer()
                    
                    Text("\(stream.viewerCount) viewers")
                        .font(.caption)
                        .foregroundColor(.secondary)
                }
            }
            .padding()
            
            Spacer()
        }
    }
}

struct YouTubePlayer: UIViewRepresentable {
    let videoId: String
    
    func makeUIView(context: Context) -> YTPlayerView {
        let playerView = YTPlayerView()
        return playerView
    }
    
    func updateUIView(_ uiView: YTPlayerView, context: Context) {
        uiView.load(withVideoId: videoId, playerVars: [
            "playsinline": 1,
            "controls": 1,
            "autohide": 1,
        ])
    }
}
```

### 5. Android App (Enhanced)

**Updated Structure** (builds on MVP):

```kotlin
// Add YouTube API integration
class BroadcastViewModel @Inject constructor(
    private val apiService: ApiService
) : ViewModel() {
    
    private var rtmpCamera: RtmpCamera? = null
    private var currentStreamId: String? = null
    
    val isStreaming = MutableStateFlow(false)
    val viewerCount = MutableStateFlow(0)
    
    fun setupCamera(context: Context, surfaceView: SurfaceView) {
        rtmpCamera = RtmpCamera(surfaceView, object : ConnectChecker {
            override fun onConnectionSuccess() {
                viewModelScope.launch {
                    // Notify backend to transition to live
                    currentStreamId?.let { apiService.startStream(it) }
                }
            }
            
            override fun onConnectionFailed(reason: String) {
                // Handle error
            }
            
            override fun onDisconnect() {
                isStreaming.value = false
            }
        })
        
        rtmpCamera?.prepareVideo(1280, 720, 30, 2500 * 1024, 0)
        rtmpCamera?.prepareAudio(128 * 1024, 44100, true)
    }
    
    suspend fun startBroadcast(title: String, description: String) {
        // Call backend API to create broadcast
        val response = apiService.createBroadcast(
            CreateBroadcastRequest(title, description)
        )
        
        currentStreamId = response.stream.id
        
        // Start RTMP streaming
        rtmpCamera?.startStream("${response.rtmpUrl}/${response.streamKey}")
        isStreaming.value = true
        
        // Start polling viewer count
        startViewerPolling()
    }
    
    fun stopBroadcast() {
        rtmpCamera?.stopStream()
        viewModelScope.launch {
            currentStreamId?.let { apiService.stopStream(it) }
        }
        isStreaming.value = false
    }
    
    private fun startViewerPolling() {
        viewModelScope.launch {
            while (isStreaming.value) {
                delay(10000) // Poll every 10 seconds
                currentStreamId?.let {
                    val stream = apiService.getStream(it)
                    viewerCount.value = stream.viewerCount
                }
            }
        }
    }
}
```

### 6. Real-time Updates (Simple Polling)

**For 100 users, simple polling is sufficient**:

```javascript
// Mobile app polling
setInterval(async () => {
  const streams = await apiClient.getStreams();
  updateStreamList(streams);
}, 15000); // Poll every 15 seconds
```

**Optional: Simple WebSocket** (if you want real-time):

```javascript
// Backend: simple WebSocket server
const WebSocket = require('ws');
const wss = new WebSocket.Server({ port: 8080 });

wss.on('connection', (ws) => {
  ws.on('message', (message) => {
    const data = JSON.parse(message);
    
    if (data.type === 'subscribe') {
      ws.streamId = data.streamId;
    }
  });
});

// Broadcast updates
function notifyStreamUpdate(streamId, data) {
  wss.clients.forEach((client) => {
    if (client.streamId === streamId) {
      client.send(JSON.stringify(data));
    }
  });
}
```

### 7. Deployment

**Single Server Setup** (DigitalOcean/Railway/Render):

**docker-compose.yml**:

```yaml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://postgres:password@postgres:5432/livestream
      - JWT_SECRET=${JWT_SECRET}
      - YOUTUBE_CLIENT_ID=${YOUTUBE_CLIENT_ID}
      - YOUTUBE_CLIENT_SECRET=${YOUTUBE_CLIENT_SECRET}
    depends_on:
      - postgres
    restart: unless-stopped
  
  postgres:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=livestream
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

volumes:
  postgres_data:
```

**Dockerfile**:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .

RUN npx prisma generate

EXPOSE 3000

CMD ["npm", "start"]
```

**Deploy to DigitalOcean** ($12/month droplet):

```bash
# On server
git clone your-repo
cd backend
docker-compose up -d

# Set up Nginx reverse proxy
sudo apt install nginx
sudo nano /etc/nginx/sites-available/default

# Nginx config:
server {
    listen 80;
    server_name your-domain.com;
    
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}

# Enable SSL
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

**Or Deploy to Railway** (even simpler):

1. Connect GitHub repo
2. Railway auto-detects and deploys
3. Automatically provides HTTPS
4. ~$5-20/month depending on usage

### 8. Monitoring (Basic)

**Sentry for Error Tracking**:

```javascript
// Backend
const Sentry = require('@sentry/node');

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
});

app.use(Sentry.Handlers.requestHandler());
app.use(Sentry.Handlers.errorHandler());
```

**Simple Logging**:

```javascript
const winston = require('winston');

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.json(),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple(),
  }));
}
```

### 9. Cost Estimation

**Monthly Costs for 100 Users**:


| Service                | Provider                       | Cost           |
| ---------------------- | ------------------------------ | -------------- |
| **VPS Server**         | DigitalOcean (4GB RAM, 2 vCPU) | $24            |
| **Database**           | Included in VPS                | $0             |
| **Domain**             | Namecheap                      | $1             |
| **SSL**                | Let's Encrypt                  | Free           |
| **Error Tracking**     | Sentry (free tier)             | Free           |
| **YouTube API**        | Google (free tier)             | Free           |
| **Push Notifications** | Firebase FCM                   | Free           |
| **Email**              | SendGrid (free tier)           | Free           |
|                        | **Total**                      | **~$25/month** |


**Alternative Hosting Options**:

- **Railway**: $5-20/month (easier deployment)
- **Render**: Free tier available, $7/month for paid
- **Fly.io**: ~$10/month
- **Heroku**: ~$16/month (hobby tier)

### 10. Development Timeline

**Realistic timeline for 1-2 developers**:


| Phase                   | Duration  | Tasks                                           |
| ----------------------- | --------- | ----------------------------------------------- |
| **Backend Setup**       | 1-2 weeks | API server, database, YouTube integration       |
| **Android Enhancement** | 2-3 weeks | YouTube API, RTMP streaming, polish             |
| **iOS Development**     | 3-4 weeks | Full iOS app from scratch                       |
| **Testing & Polish**    | 1-2 weeks | Bug fixes, UX improvements                      |
| **Deployment**          | 1 week    | Server setup, domain, SSL, app store submission |
|                         | **Total** | **8-12 weeks**                                  |


### 11. Key Differences from MVP


| Feature               | MVP                    | 100-User Version         |
| --------------------- | ---------------------- | ------------------------ |
| **iOS Support**       | ❌ None                 | ✅ Full native app        |
| **In-App Streaming**  | ❌ Launches YouTube app | ✅ Stream without leaving |
| **YouTube API**       | ❌ Just stores URLs     | ✅ Full control via API   |
| **Backend**           | ❌ Firebase only        | ✅ Custom Node.js API     |
| **Stream Control**    | ❌ Manual in YouTube    | ✅ Programmatic control   |
| **User Management**   | ✅ Basic                | ✅ Full profiles          |
| **Real-time Updates** | ✅ Firestore            | ✅ Polling or WebSocket   |
| **Cost**              | Free                   | ~$25/month               |
| **Scalability**       | 5-10 users             | ~100 users               |


### 12. Future Scaling Path (When You Need More)

**When you hit 100+ concurrent users and need to scale further**:

1. **Separate database server** ($15-30/month)
2. **Add Redis caching** ($10-20/month)
3. **CDN for assets** (Cloudflare free tier)
4. **Load balancer + 2nd API server** (+$24/month)
5. **Managed database** (AWS RDS, +$50/month)
6. **Auto-scaling** (Kubernetes or AWS ECS)

This would bring you to the ~500-1000 user range at ~$150-200/month.

### 13. Security Essentials

**Must-haves for 100 users**:

```javascript
// Rate limiting
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per window
});

app.use('/api/', limiter);

// Helmet for security headers
const helmet = require('helmet');
app.use(helmet());

// CORS
const cors = require('cors');
app.use(cors({
  origin: process.env.ALLOWED_ORIGINS.split(','),
  credentials: true,
}));

// Input validation
const { body, validationResult } = require('express-validator');

app.post('/api/streams',
  body('title').trim().isLength({ min: 3, max: 100 }).escape(),
  body('description').trim().isLength({ max: 1000 }).escape(),
  async (req, res) => {
    const errors = validationResult(req);
    if (!errors.isEmpty()) {
      return res.status(400).json({ errors: errors.array() });
    }
    // Process request
  }
);

// Encrypt sensitive data
const crypto = require('crypto');

function encrypt(text) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-cbc', Buffer.from(process.env.ENCRYPTION_KEY, 'hex'), iv);
  let encrypted = cipher.update(text);
  encrypted = Buffer.concat([encrypted, cipher.final()]);
  return iv.toString('hex') + ':' + encrypted.toString('hex');
}

function decrypt(text) {
  const textParts = text.split(':');
  const iv = Buffer.from(textParts.shift(), 'hex');
  const encryptedText = Buffer.from(textParts.join(':'), 'hex');
  const decipher = crypto.createDecipheriv('aes-256-cbc', Buffer.from(process.env.ENCRYPTION_KEY, 'hex'), iv);
  let decrypted = decipher.update(encryptedText);
  decrypted = Buffer.concat([decrypted, decipher.final()]);
  return decrypted.toString();
}
```

### 14. Environment Variables

```bash
# .env
NODE_ENV=production
PORT=3000

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/livestream

# JWT
JWT_SECRET=your-random-256-bit-secret
JWT_EXPIRES_IN=7d

# YouTube API
YOUTUBE_CLIENT_ID=xxx.apps.googleusercontent.com
YOUTUBE_CLIENT_SECRET=xxx
YOUTUBE_REDIRECT_URI=https://your-domain.com/api/auth/youtube/callback

# Encryption
ENCRYPTION_KEY=64-character-hex-string

# Sentry (optional)
SENTRY_DSN=https://xxx@sentry.io/xxx

# CORS
ALLOWED_ORIGINS=https://your-domain.com,capacitor://localhost,http://localhost
```

## Summary

**This 100-user plan provides**:

✅ **iOS + Android apps** with in-app YouTube streaming
✅ **Simple backend** (single Node.js server + PostgreSQL)
✅ **YouTube API integration** for full control
✅ **Low cost** (~$25/month)
✅ **Manageable complexity** (1-2 developers can handle it)
✅ **Clear scaling path** when you need to grow beyond 100 users

**Development time**: 8-12 weeks with 1-2 developers

**Perfect for**: Testing the market, early adopters, beta phase, sports teams/leagues with 50-200 people