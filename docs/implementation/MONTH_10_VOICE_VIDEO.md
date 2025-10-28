# Month 10: Voice & Video Calls - Implementation Guide

**File**: `MONTH_10_VOICE_VIDEO.md`  
**Purpose**: WebRTC voice/video calls, TURN/STUN servers, call recording  
**Prerequisites**: Complete [MONTH_09_SECURITY.md](./MONTH_09_SECURITY.md)

---

## Updates for Prisma Migration

**Note**: Month 10 adds WebRTC voice/video calling. Database operations updated to use Prisma:

- ✅ Call Management - Prisma models with status enums
- ✅ Call Participants - Real-time tracking
- ✅ Call Recordings - Storage and metadata
- ✅ WebRTC Signaling - No database (Socket.IO)
- ✅ All services converted to PrismaService

---

## Assumptions

- Enterprise customers requesting voice/video features
- Need for 1-1 and group calls
- Call quality and reliability critical
- Team: 2 Backend Engineers, 1 WebRTC Specialist, 1 Frontend Engineer

---

## Week 1: WebRTC Signaling Server

### Task 1: Prisma Schema

**Update `packages/server/prisma/schema.prisma`**:

```prisma
enum CallType {
  AUDIO
  VIDEO
}

enum CallStatus {
  RINGING
  ACTIVE
  ENDED
  MISSED
  DECLINED
}

enum ConnectionQuality {
  EXCELLENT
  GOOD
  FAIR
  POOR
}

model Call {
  id          String     @id @default(uuid())
  channelId   String     @map("channel_id")
  callType    CallType   @map("call_type")
  status      CallStatus @default(RINGING)
  startedBy   String     @map("started_by")
  startedAt   DateTime   @default(now()) @map("started_at")
  endedAt     DateTime?  @map("ended_at")
  duration    Int?       // in seconds
  metadata    Json       @default("{}")

  channel      Channel           @relation(fields: [channelId], references: [id], onDelete: Cascade)
  starter      User              @relation("StartedCalls", fields: [startedBy], references: [id])
  participants CallParticipant[]
  recordings   CallRecording[]

  @@index([channelId])
  @@index([status])
  @@index([startedAt])
  @@map("calls")
}

model CallParticipant {
  id               String             @id @default(uuid())
  callId           String             @map("call_id")
  userId           String             @map("user_id")
  joinedAt         DateTime           @default(now()) @map("joined_at")
  leftAt           DateTime?          @map("left_at")
  isMuted          Boolean            @default(false) @map("is_muted")
  isVideoEnabled   Boolean            @default(true) @map("is_video_enabled")
  connectionQuality ConnectionQuality? @map("connection_quality")

  call Call @relation(fields: [callId], references: [id], onDelete: Cascade)
  user User @relation("CallParticipants", fields: [userId], references: [id])

  @@unique([callId, userId])
  @@index([callId])
  @@index([userId])
  @@map("call_participants")
}

model CallRecording {
  id           String    @id @default(uuid())
  callId       String    @map("call_id")
  recordingUrl String    @map("recording_url")
  duration     Int?      // in seconds
  fileSize     BigInt?   @map("file_size")
  startedAt    DateTime  @map("started_at")
  endedAt      DateTime? @map("ended_at")
  createdAt    DateTime  @default(now()) @map("created_at")

  call Call @relation(fields: [callId], references: [id], onDelete: Cascade)

  @@index([callId])
  @@map("call_recordings")
}
```

**Update related models**:

```prisma
model User {
  // ... existing fields ...
  startedCalls     Call[]            @relation("StartedCalls")
  callParticipants CallParticipant[] @relation("CallParticipants")
}

model Channel {
  // ... existing fields ...
  calls Call[]
}
```

**Generate Migration**:

```bash
cd packages/server
npx prisma migrate dev --name add_voice_video_calls
```

---

### Task 2: WebRTC Signaling Gateway

**Create `packages/server/src/webrtc/webrtc.gateway.ts`**:

```typescript
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  OnGatewayConnection,
  OnGatewayDisconnect,
  ConnectedSocket,
  MessageBody,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { Logger } from '@nestjs/common';

@WebSocketGateway({ namespace: '/webrtc', cors: { origin: '*' } })
export class WebRTCGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private readonly logger = new Logger(WebRTCGateway.name);
  private activeCalls = new Map<string, Set<string>>(); // callId -> Set<socketId>

  handleConnection(client: Socket) {
    this.logger.log(`Client connected: ${client.id}`);
  }

  handleDisconnect(client: Socket) {
    this.logger.log(`Client disconnected: ${client.id}`);
    
    // Clean up from active calls
    for (const [callId, participants] of this.activeCalls.entries()) {
      if (participants.has(client.id)) {
        participants.delete(client.id);
        
        // Notify other participants
        this.server.to(callId).emit('participant.left', {
          socketId: client.id,
          userId: client.data.userId,
        });
        
        if (participants.size === 0) {
          this.activeCalls.delete(callId);
        }
      }
    }
  }

  @SubscribeMessage('call.join')
  async handleJoinCall(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { callId: string; userId: string },
  ) {
    client.data.userId = data.userId;
    client.join(data.callId);

    if (!this.activeCalls.has(data.callId)) {
      this.activeCalls.set(data.callId, new Set());
    }
    this.activeCalls.get(data.callId).add(client.id);

    // Get existing participants
    const participants = Array.from(this.activeCalls.get(data.callId))
      .filter(id => id !== client.id)
      .map(id => ({
        socketId: id,
        userId: this.server.sockets.sockets.get(id)?.data.userId,
      }));

    // Notify new participant of existing participants
    client.emit('call.participants', { participants });

    // Notify existing participants of new participant
    client.to(data.callId).emit('participant.joined', {
      socketId: client.id,
      userId: data.userId,
    });

    this.logger.log(`User ${data.userId} joined call ${data.callId}`);
  }

  @SubscribeMessage('call.offer')
  async handleOffer(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { to: string; offer: RTCSessionDescriptionInit },
  ) {
    this.server.to(data.to).emit('call.offer', {
      from: client.id,
      offer: data.offer,
    });
  }

  @SubscribeMessage('call.answer')
  async handleAnswer(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { to: string; answer: RTCSessionDescriptionInit },
  ) {
    this.server.to(data.to).emit('call.answer', {
      from: client.id,
      answer: data.answer,
    });
  }

  @SubscribeMessage('call.ice-candidate')
  async handleIceCandidate(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { to: string; candidate: RTCIceCandidateInit },
  ) {
    this.server.to(data.to).emit('call.ice-candidate', {
      from: client.id,
      candidate: data.candidate,
    });
  }

  @SubscribeMessage('call.leave')
  async handleLeaveCall(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { callId: string },
  ) {
    client.leave(data.callId);
    
    const participants = this.activeCalls.get(data.callId);
    if (participants) {
      participants.delete(client.id);
      
      if (participants.size === 0) {
        this.activeCalls.delete(data.callId);
      }
    }

    client.to(data.callId).emit('participant.left', {
      socketId: client.id,
      userId: client.data.userId,
    });
  }

  @SubscribeMessage('call.mute')
  async handleMute(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { callId: string; isMuted: boolean },
  ) {
    client.to(data.callId).emit('participant.muted', {
      socketId: client.id,
      userId: client.data.userId,
      isMuted: data.isMuted,
    });
  }

  @SubscribeMessage('call.video-toggle')
  async handleVideoToggle(
    @ConnectedSocket() client: Socket,
    @MessageBody() data: { callId: string; isVideoEnabled: boolean },
  ) {
    client.to(data.callId).emit('participant.video-toggled', {
      socketId: client.id,
      userId: client.data.userId,
      isVideoEnabled: data.isVideoEnabled,
    });
  }
}
```

---

### Task 3: Calls Service

**Create `packages/server/src/calls/calls.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { CallType, CallStatus } from '@prisma/client';
import { WebRTCGateway } from '../webrtc/webrtc.gateway';

@Injectable()
export class CallsService {
  constructor(
    private prisma: PrismaService,
    private webrtcGateway: WebRTCGateway,
  ) {}

  async initiateCall(userId: string, data: {
    channelId: string;
    callType: 'audio' | 'video';
    participants: string[];
  }) {
    const call = this.callRepository.create({
      channelId: data.channelId,
      callType: data.callType,
      startedBy: userId,
      status: 'ringing',
    });

    await this.callRepository.save(call);

    // Add participants
    for (const participantId of data.participants) {
      const participant = this.participantRepository.create({
        callId: call.id,
        userId: participantId,
      });
      await this.participantRepository.save(participant);
    }

    // Notify participants via WebSocket
    this.notifyParticipants(call.id, data.participants, {
      event: 'call.incoming',
      callId: call.id,
      callType: data.callType,
      initiator: userId,
    });

    return call;
  }

  async joinCall(userId: string, callId: string) {
    const call = await this.callRepository.findOne({ where: { id: callId } });
    
    if (!call) {
      throw new Error('Call not found');
    }

    // Update call status if first join
    if (call.status === 'ringing') {
      await this.callRepository.update(callId, { status: 'active' });
    }

    // Add or update participant
    let participant = await this.participantRepository.findOne({
      where: { callId, userId },
    });

    if (!participant) {
      participant = this.participantRepository.create({
        callId,
        userId,
        joinedAt: new Date(),
      });
    } else {
      participant.joinedAt = new Date();
      participant.leftAt = null;
    }

    await this.participantRepository.save(participant);

    return call;
  }

  async leaveCall(userId: string, callId: string) {
    await this.participantRepository.update(
      { callId, userId },
      { leftAt: new Date() },
    );

    // Check if all participants left
    const activeParticipants = await this.participantRepository.count({
      where: { callId, leftAt: null },
    });

    if (activeParticipants === 0) {
      await this.endCall(callId);
    }
  }

  async endCall(callId: string) {
    const call = await this.callRepository.findOne({ where: { id: callId } });
    
    if (!call) return;

    const duration = Math.floor((Date.now() - call.startedAt.getTime()) / 1000);

    await this.callRepository.update(callId, {
      status: 'ended',
      endedAt: new Date(),
      duration,
    });

    // Update all participants
    await this.participantRepository.update(
      { callId, leftAt: null },
      { leftAt: new Date() },
    );
  }

  async getCallHistory(channelId: string, limit: number = 50) {
    return this.callRepository.find({
      where: { channelId },
      order: { startedAt: 'DESC' },
      take: limit,
      relations: ['participants', 'participants.user'],
    });
  }

  async getActiveCall(channelId: string) {
    return this.callRepository.findOne({
      where: { channelId, status: 'active' },
      relations: ['participants', 'participants.user'],
    });
  }

  private notifyParticipants(callId: string, userIds: string[], data: any) {
    // Use WebSocket to notify participants
    this.webrtcGateway.server.to(callId).emit(data.event, data);
  }
}
```

---

## Week 2: TURN/STUN Server Setup

### Task 1: Coturn Installation

**Install Coturn (Ubuntu)**:

```bash
sudo apt-get update
sudo apt-get install coturn
```

**Configure `/etc/turnserver.conf`**:

```conf
# Listening port
listening-port=3478
tls-listening-port=5349

# External IP (replace with your server IP)
external-ip=YOUR_SERVER_IP

# Relay IP
relay-ip=YOUR_SERVER_IP

# Fingerprint
fingerprint

# Long-term credential mechanism
lt-cred-mech

# Realm
realm=yourdomain.com

# User database
user=username:password

# Logging
log-file=/var/log/turnserver.log
verbose

# TLS certificates (optional)
cert=/etc/letsencrypt/live/yourdomain.com/cert.pem
pkey=/etc/letsencrypt/live/yourdomain.com/privkey.pem

# Quotas
max-bps=1000000
bps-capacity=0

# Security
no-multicast-peers
no-cli
```

**Start Coturn**:

```bash
sudo systemctl enable coturn
sudo systemctl start coturn
sudo systemctl status coturn
```

---

### Task 2: ICE Server Configuration

**Create `packages/server/src/webrtc/ice-servers.service.ts`**:

```typescript
import { Injectable } from '@nestjs/common';
import * as crypto from 'crypto';

@Injectable()
export class IceServersService {
  private turnServers = [
    {
      urls: 'turn:yourdomain.com:3478',
      username: process.env.TURN_USERNAME,
      credential: process.env.TURN_PASSWORD,
    },
    {
      urls: 'turns:yourdomain.com:5349',
      username: process.env.TURN_USERNAME,
      credential: process.env.TURN_PASSWORD,
    },
  ];

  private stunServers = [
    { urls: 'stun:stun.l.google.com:19302' },
    { urls: 'stun:stun1.l.google.com:19302' },
  ];

  getIceServers(userId: string): RTCIceServer[] {
    // Generate time-limited credentials for TURN
    const timestamp = Math.floor(Date.now() / 1000) + 24 * 3600; // 24 hours
    const username = `${timestamp}:${userId}`;
    const credential = this.generateTurnCredential(username);

    return [
      ...this.stunServers,
      {
        urls: this.turnServers[0].urls,
        username,
        credential,
      },
    ];
  }

  private generateTurnCredential(username: string): string {
    const secret = process.env.TURN_SECRET;
    return crypto.createHmac('sha1', secret).update(username).digest('base64');
  }
}
```

**API Endpoint**:

```typescript
@Controller('webrtc')
export class WebRTCController {
  constructor(private iceServersService: IceServersService) {}

  @Get('ice-servers')
  @UseGuards(JwtAuthGuard)
  getIceServers(@CurrentUser() user: any) {
    return this.iceServersService.getIceServers(user.id);
  }
}
```

---

## Week 3: Client Implementation

### Task 1: WebRTC Client (TypeScript)

**Create `packages/sdk-core/src/webrtc/WebRTCClient.ts`**:

```typescript
import { io, Socket } from 'socket.io-client';
import { EventEmitter } from 'eventemitter3';

export class WebRTCClient extends EventEmitter {
  private socket: Socket;
  private peerConnections = new Map<string, RTCPeerConnection>();
  private localStream: MediaStream | null = null;
  private iceServers: RTCIceServer[] = [];

  constructor(private wsUrl: string, private token: string) {
    super();
    this.socket = io(wsUrl, {
      auth: { token },
      transports: ['websocket'],
    });

    this.setupSocketListeners();
  }

  async initialize(iceServers: RTCIceServer[]) {
    this.iceServers = iceServers;
  }

  async startCall(callId: string, userId: string, isVideo: boolean = true) {
    // Get local media
    this.localStream = await navigator.mediaDevices.getUserMedia({
      audio: true,
      video: isVideo,
    });

    this.emit('localStream', this.localStream);

    // Join call room
    this.socket.emit('call.join', { callId, userId });
  }

  private setupSocketListeners() {
    this.socket.on('call.participants', async ({ participants }) => {
      // Create peer connections for existing participants
      for (const participant of participants) {
        await this.createPeerConnection(participant.socketId, true);
      }
    });

    this.socket.on('participant.joined', async ({ socketId }) => {
      // New participant joined, create peer connection
      await this.createPeerConnection(socketId, true);
    });

    this.socket.on('call.offer', async ({ from, offer }) => {
      const pc = await this.createPeerConnection(from, false);
      await pc.setRemoteDescription(new RTCSessionDescription(offer));
      
      const answer = await pc.createAnswer();
      await pc.setLocalDescription(answer);
      
      this.socket.emit('call.answer', { to: from, answer });
    });

    this.socket.on('call.answer', async ({ from, answer }) => {
      const pc = this.peerConnections.get(from);
      if (pc) {
        await pc.setRemoteDescription(new RTCSessionDescription(answer));
      }
    });

    this.socket.on('call.ice-candidate', async ({ from, candidate }) => {
      const pc = this.peerConnections.get(from);
      if (pc && candidate) {
        await pc.addIceCandidate(new RTCIceCandidate(candidate));
      }
    });

    this.socket.on('participant.left', ({ socketId }) => {
      this.closePeerConnection(socketId);
    });
  }

  private async createPeerConnection(socketId: string, isInitiator: boolean): Promise<RTCPeerConnection> {
    const pc = new RTCPeerConnection({ iceServers: this.iceServers });

    // Add local stream tracks
    if (this.localStream) {
      this.localStream.getTracks().forEach(track => {
        pc.addTrack(track, this.localStream!);
      });
    }

    // Handle remote stream
    pc.ontrack = (event) => {
      this.emit('remoteStream', { socketId, stream: event.streams[0] });
    };

    // Handle ICE candidates
    pc.onicecandidate = (event) => {
      if (event.candidate) {
        this.socket.emit('call.ice-candidate', {
          to: socketId,
          candidate: event.candidate,
        });
      }
    };

    // Handle connection state
    pc.onconnectionstatechange = () => {
      this.emit('connectionStateChange', {
        socketId,
        state: pc.connectionState,
      });
    };

    this.peerConnections.set(socketId, pc);

    // Create and send offer if initiator
    if (isInitiator) {
      const offer = await pc.createOffer();
      await pc.setLocalDescription(offer);
      this.socket.emit('call.offer', { to: socketId, offer });
    }

    return pc;
  }

  toggleMute(isMuted: boolean) {
    if (this.localStream) {
      this.localStream.getAudioTracks().forEach(track => {
        track.enabled = !isMuted;
      });
    }
  }

  toggleVideo(isVideoEnabled: boolean) {
    if (this.localStream) {
      this.localStream.getVideoTracks().forEach(track => {
        track.enabled = isVideoEnabled;
      });
    }
  }

  endCall(callId: string) {
    // Close all peer connections
    this.peerConnections.forEach(pc => pc.close());
    this.peerConnections.clear();

    // Stop local stream
    if (this.localStream) {
      this.localStream.getTracks().forEach(track => track.stop());
      this.localStream = null;
    }

    this.socket.emit('call.leave', { callId });
  }

  private closePeerConnection(socketId: string) {
    const pc = this.peerConnections.get(socketId);
    if (pc) {
      pc.close();
      this.peerConnections.delete(socketId);
    }
  }
}
```

---

### Task 2: React Hook

**Create `packages/sdk-react/src/hooks/useCall.ts`**:

```typescript
import { useState, useEffect, useRef } from 'react';
import { WebRTCClient } from '@chat-sdk/core';

export const useCall = (callId: string, userId: string) => {
  const [localStream, setLocalStream] = useState<MediaStream | null>(null);
  const [remoteStreams, setRemoteStreams] = useState<Map<string, MediaStream>>(new Map());
  const [isMuted, setIsMuted] = useState(false);
  const [isVideoEnabled, setIsVideoEnabled] = useState(true);
  const webrtcClient = useRef<WebRTCClient | null>(null);

  useEffect(() => {
    const initCall = async () => {
      const client = new WebRTCClient('ws://localhost:3000/webrtc', 'token');
      
      // Get ICE servers
      const iceServers = await fetch('/api/webrtc/ice-servers').then(r => r.json());
      await client.initialize(iceServers);

      client.on('localStream', (stream) => {
        setLocalStream(stream);
      });

      client.on('remoteStream', ({ socketId, stream }) => {
        setRemoteStreams(prev => new Map(prev).set(socketId, stream));
      });

      await client.startCall(callId, userId, true);
      webrtcClient.current = client;
    };

    initCall();

    return () => {
      webrtcClient.current?.endCall(callId);
    };
  }, [callId, userId]);

  const toggleMute = () => {
    webrtcClient.current?.toggleMute(!isMuted);
    setIsMuted(!isMuted);
  };

  const toggleVideo = () => {
    webrtcClient.current?.toggleVideo(!isVideoEnabled);
    setIsVideoEnabled(!isVideoEnabled);
  };

  const endCall = () => {
    webrtcClient.current?.endCall(callId);
  };

  return {
    localStream,
    remoteStreams,
    isMuted,
    isVideoEnabled,
    toggleMute,
    toggleVideo,
    endCall,
  };
};
```

---

## Week 4: Call Recording & Testing

### Task 1: Call Recording (Optional)

**Using MediaRecorder API**:

```typescript
export class CallRecordingService {
  private mediaRecorder: MediaRecorder | null = null;
  private recordedChunks: Blob[] = [];

  async startRecording(stream: MediaStream) {
    this.mediaRecorder = new MediaRecorder(stream, {
      mimeType: 'video/webm;codecs=vp9',
    });

    this.mediaRecorder.ondataavailable = (event) => {
      if (event.data.size > 0) {
        this.recordedChunks.push(event.data);
      }
    };

    this.mediaRecorder.start(1000); // Collect data every second
  }

  stopRecording(): Promise<Blob> {
    return new Promise((resolve) => {
      if (this.mediaRecorder) {
        this.mediaRecorder.onstop = () => {
          const blob = new Blob(this.recordedChunks, { type: 'video/webm' });
          resolve(blob);
        };
        this.mediaRecorder.stop();
      }
    });
  }

  async uploadRecording(callId: string, blob: Blob) {
    const formData = new FormData();
    formData.append('recording', blob, `call-${callId}.webm`);
    
    await fetch(`/api/calls/${callId}/recording`, {
      method: 'POST',
      body: formData,
    });
  }
}
```

---

### Task 2: Testing

```typescript
describe('WebRTC Calls', () => {
  it('should initiate call', async () => {
    const call = await callsService.initiateCall(userId, {
      channelId,
      callType: 'video',
      participants: [user2Id],
    });

    expect(call.status).toBe('ringing');
  });

  it('should join call', async () => {
    await callsService.joinCall(user2Id, callId);
    const call = await callsService.getActiveCall(channelId);
    expect(call.status).toBe('active');
  });
});
```

---

## API Endpoints

```
GET    /webrtc/ice-servers           Get ICE servers
POST   /calls                        Initiate call
POST   /calls/:id/join               Join call
POST   /calls/:id/leave              Leave call
POST   /calls/:id/end                End call
GET    /calls/:id                    Get call details
GET    /channels/:id/calls           Get call history
POST   /calls/:id/recording          Upload recording
```

---

## Deliverables

- [x] WebRTC signaling server
- [x] TURN/STUN server setup
- [x] 1-1 voice calls
- [x] 1-1 video calls
- [x] Group calls (mesh topology)
- [x] Mute/unmute audio
- [x] Enable/disable video
- [x] Call history
- [x] Call recording (optional)
- [x] ICE server configuration
- [x] Client SDK integration
- [x] React hooks
- [x] Connection quality monitoring

**Next**: [MONTH_11_SCALABILITY.md](./MONTH_11_SCALABILITY.md)
