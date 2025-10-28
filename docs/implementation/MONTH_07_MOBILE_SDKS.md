# Month 7: Mobile SDKs - Implementation Guide

**File**: `MONTH_07_MOBILE_SDKS.md`  
**Purpose**: Build native iOS, Android, and React Native SDKs  
**Prerequisites**: Complete [MONTH_06_ANALYTICS_WEBHOOKS.md](./MONTH_06_ANALYTICS_WEBHOOKS.md)

---

## Assumptions

- Core SDK (TypeScript) is stable and tested
- Mobile developers available (1 iOS, 1 Android, 1 React Native)
- Push notification infrastructure ready (FCM, APNS)
- Target: iOS 14+, Android 8+ (API 26+)

---

## Week 1-2: iOS SDK (Swift)

### Task 1: iOS Project Setup

**Create Swift Package**:

```bash
mkdir -p packages/sdk-ios
cd packages/sdk-ios
swift package init --type library --name ChatSDK
```

**Package.swift**:

```swift
// swift-tools-version:5.9
import PackageDescription

let package = Package(
    name: "ChatSDK",
    platforms: [
        .iOS(.v14),
        .macOS(.v11)
    ],
    products: [
        .library(
            name: "ChatSDK",
            targets: ["ChatSDK"]),
    ],
    dependencies: [
        .package(url: "https://github.com/socketio/socket.io-client-swift", from: "16.0.0"),
        .package(url: "https://github.com/Alamofire/Alamofire.git", from: "5.8.0")
    ],
    targets: [
        .target(
            name: "ChatSDK",
            dependencies: [
                .product(name: "SocketIO", package: "socket.io-client-swift"),
                "Alamofire"
            ]),
        .testTarget(
            name: "ChatSDKTests",
            dependencies: ["ChatSDK"]),
    ]
)
```

---

### Task 2: Core Models

**Create `Sources/ChatSDK/Models/User.swift`**:

```swift
import Foundation

public struct User: Codable, Identifiable {
    public let id: String
    public let email: String?
    public let username: String?
    public let displayName: String?
    public let avatarUrl: String?
    public let userType: UserType
    public var isOnline: Bool?
    
    public enum UserType: String, Codable {
        case regular
        case anonymous
        case guest
    }
    
    public init(id: String, email: String? = nil, username: String? = nil, 
                displayName: String? = nil, avatarUrl: String? = nil, 
                userType: UserType = .regular, isOnline: Bool? = nil) {
        self.id = id
        self.email = email
        self.username = username
        self.displayName = displayName
        self.avatarUrl = avatarUrl
        self.userType = userType
        self.isOnline = isOnline
    }
}
```

**Create `Sources/ChatSDK/Models/Channel.swift`**:

```swift
import Foundation

public struct Channel: Codable, Identifiable {
    public let id: String
    public let type: ChannelType
    public let name: String?
    public let description: String?
    public let isDistinct: Bool?
    public let isFrozen: Bool?
    public var members: [ChannelMember]?
    public var lastMessage: Message?
    public let createdAt: Date
    public let updatedAt: Date
    
    public enum ChannelType: String, Codable {
        case direct
        case group
        case `public`
    }
}

public struct ChannelMember: Codable {
    public let userId: String
    public let role: String
    public let joinedAt: Date
}
```

**Create `Sources/ChatSDK/Models/Message.swift`**:

```swift
import Foundation

public struct Message: Codable, Identifiable {
    public let id: String
    public let channelId: String
    public let userId: String
    public let text: String?
    public let type: MessageType
    public var status: MessageStatus
    public let attachments: [Attachment]?
    public let mentions: [String]?
    public let createdAt: Date
    public let updatedAt: Date
    
    public enum MessageType: String, Codable {
        case text
        case image
        case video
        case file
        case system
    }
    
    public enum MessageStatus: String, Codable {
        case pending
        case sent
        case delivered
        case failed
    }
}

public struct Attachment: Codable, Identifiable {
    public let id: String
    public let type: String
    public let url: String
    public let fileName: String?
    public let fileSize: Int?
}
```

---

### Task 3: API Client

**Create `Sources/ChatSDK/Network/APIClient.swift`**:

```swift
import Foundation
import Alamofire

public class APIClient {
    private let baseURL: String
    private let session: Session
    private var token: String?
    
    public init(baseURL: String) {
        self.baseURL = baseURL
        
        let configuration = URLSessionConfiguration.default
        configuration.timeoutIntervalForRequest = 30
        self.session = Session(configuration: configuration)
    }
    
    public func setAuthToken(_ token: String) {
        self.token = token
    }
    
    private var headers: HTTPHeaders {
        var headers = HTTPHeaders()
        if let token = token {
            headers.add(.authorization(bearerToken: token))
        }
        headers.add(.contentType("application/json"))
        return headers
    }
    
    // MARK: - Channels
    
    public func createChannel(type: String, name: String?, members: [String]?) async throws -> Channel {
        let parameters: [String: Any] = [
            "type": type,
            "name": name as Any,
            "members": members as Any
        ]
        
        return try await request(
            path: "/channels",
            method: .post,
            parameters: parameters
        )
    }
    
    public func getChannels() async throws -> [Channel] {
        return try await request(path: "/channels", method: .get)
    }
    
    public func getChannel(id: String) async throws -> Channel {
        return try await request(path: "/channels/\(id)", method: .get)
    }
    
    // MARK: - Messages
    
    public func sendMessage(channelId: String, text: String) async throws -> Message {
        let parameters: [String: Any] = [
            "channelId": channelId,
            "text": text
        ]
        
        return try await request(
            path: "/messages",
            method: .post,
            parameters: parameters
        )
    }
    
    public func getMessages(channelId: String, limit: Int = 50) async throws -> [Message] {
        return try await request(
            path: "/channels/\(channelId)/messages",
            method: .get,
            parameters: ["limit": limit]
        )
    }
    
    // MARK: - Generic Request
    
    private func request<T: Decodable>(
        path: String,
        method: HTTPMethod,
        parameters: Parameters? = nil
    ) async throws -> T {
        let url = baseURL + path
        
        return try await withCheckedThrowingContinuation { continuation in
            session.request(
                url,
                method: method,
                parameters: parameters,
                encoding: method == .get ? URLEncoding.default : JSONEncoding.default,
                headers: headers
            )
            .validate()
            .responseDecodable(of: T.self) { response in
                switch response.result {
                case .success(let value):
                    continuation.resume(returning: value)
                case .failure(let error):
                    continuation.resume(throwing: error)
                }
            }
        }
    }
}
```

---

### Task 4: WebSocket Manager

**Create `Sources/ChatSDK/Network/WebSocketManager.swift`**:

```swift
import Foundation
import SocketIO

public protocol WebSocketDelegate: AnyObject {
    func webSocketDidConnect()
    func webSocketDidDisconnect(reason: String)
    func webSocketDidReceiveMessage(_ message: Message)
    func webSocketDidReceiveTyping(userId: String, channelId: String, isTyping: Bool)
}

public class WebSocketManager {
    private let manager: SocketManager
    private let socket: SocketIOClient
    public weak var delegate: WebSocketDelegate?
    
    public init(url: String, token: String) {
        let config: SocketIOClientConfiguration = [
            .log(false),
            .compress,
            .reconnects(true),
            .reconnectAttempts(5),
            .reconnectWait(2),
            .forceWebsockets(true),
            .extraHeaders(["Authorization": "Bearer \(token)"])
        ]
        
        self.manager = SocketManager(socketURL: URL(string: url)!, config: config)
        self.socket = manager.defaultSocket
        
        setupEventHandlers()
    }
    
    public func connect() {
        socket.connect()
    }
    
    public func disconnect() {
        socket.disconnect()
    }
    
    public func joinChannel(_ channelId: String) {
        socket.emit("channel.join", ["channelId": channelId])
    }
    
    public func leaveChannel(_ channelId: String) {
        socket.emit("channel.leave", ["channelId": channelId])
    }
    
    public func sendTyping(channelId: String, isTyping: Bool) {
        let event = isTyping ? "typing.start" : "typing.stop"
        socket.emit(event, ["channelId": channelId])
    }
    
    private func setupEventHandlers() {
        socket.on(clientEvent: .connect) { [weak self] _, _ in
            self?.delegate?.webSocketDidConnect()
        }
        
        socket.on(clientEvent: .disconnect) { [weak self] data, _ in
            let reason = data.first as? String ?? "Unknown"
            self?.delegate?.webSocketDidDisconnect(reason: reason)
        }
        
        socket.on("message.new") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let jsonData = try? JSONSerialization.data(withJSONObject: dict),
                  let message = try? JSONDecoder().decode(Message.self, from: jsonData) else {
                return
            }
            self?.delegate?.webSocketDidReceiveMessage(message)
        }
        
        socket.on("typing.start") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let userId = dict["userId"] as? String,
                  let channelId = dict["channelId"] as? String else {
                return
            }
            self?.delegate?.webSocketDidReceiveTyping(userId: userId, channelId: channelId, isTyping: true)
        }
        
        socket.on("typing.stop") { [weak self] data, _ in
            guard let dict = data.first as? [String: Any],
                  let userId = dict["userId"] as? String,
                  let channelId = dict["channelId"] as? String else {
                return
            }
            self?.delegate?.webSocketDidReceiveTyping(userId: userId, channelId: channelId, isTyping: false)
        }
    }
}
```

---

### Task 5: Chat Client

**Create `Sources/ChatSDK/ChatClient.swift`**:

```swift
import Foundation

public class ChatClient {
    private let apiClient: APIClient
    private var wsManager: WebSocketManager?
    public private(set) var currentUser: User?
    
    public weak var delegate: WebSocketDelegate? {
        didSet {
            wsManager?.delegate = delegate
        }
    }
    
    public init(apiURL: String, wsURL: String? = nil, token: String? = nil) {
        self.apiClient = APIClient(baseURL: apiURL)
        
        if let token = token {
            self.apiClient.setAuthToken(token)
            
            let socketURL = wsURL ?? apiURL.replacingOccurrences(of: "http", with: "ws")
            self.wsManager = WebSocketManager(url: socketURL, token: token)
        }
    }
    
    public func connect() {
        wsManager?.connect()
    }
    
    public func disconnect() {
        wsManager?.disconnect()
    }
    
    // MARK: - Channels
    
    public func createChannel(type: String, name: String?, members: [String]? = nil) async throws -> Channel {
        return try await apiClient.createChannel(type: type, name: name, members: members)
    }
    
    public func getChannels() async throws -> [Channel] {
        return try await apiClient.getChannels()
    }
    
    public func joinChannel(_ channelId: String) {
        wsManager?.joinChannel(channelId)
    }
    
    // MARK: - Messages
    
    public func sendMessage(channelId: String, text: String) async throws -> Message {
        return try await apiClient.sendMessage(channelId: channelId, text: text)
    }
    
    public func getMessages(channelId: String, limit: Int = 50) async throws -> [Message] {
        return try await apiClient.getMessages(channelId: channelId, limit: limit)
    }
    
    // MARK: - Typing
    
    public func startTyping(in channelId: String) {
        wsManager?.sendTyping(channelId: channelId, isTyping: true)
    }
    
    public func stopTyping(in channelId: String) {
        wsManager?.sendTyping(channelId: channelId, isTyping: false)
    }
}
```

---

## Week 3: Android SDK (Kotlin)

### Task 1: Android Project Setup

**Create `packages/sdk-android/build.gradle.kts`**:

```kotlin
plugins {
    id("com.android.library")
    id("org.jetbrains.kotlin.android")
    id("maven-publish")
}

android {
    namespace = "com.chatsdk.android"
    compileSdk = 34

    defaultConfig {
        minSdk = 26
        targetSdk = 34
    }

    compileOptions {
        sourceCompatibility = JavaVersion.VERSION_17
        targetCompatibility = JavaVersion.VERSION_17
    }
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
    implementation("com.squareup.retrofit2:retrofit:2.9.0")
    implementation("com.squareup.retrofit2:converter-gson:2.9.0")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("io.socket:socket.io-client:2.1.0")
    
    testImplementation("junit:junit:4.13.2")
    testImplementation("org.jetbrains.kotlinx:kotlinx-coroutines-test:1.7.3")
}
```

---

### Task 2: Models

**Create `sdk-android/src/main/java/com/chatsdk/models/User.kt`**:

```kotlin
package com.chatsdk.models

import com.google.gson.annotations.SerializedName

data class User(
    val id: String,
    val email: String? = null,
    val username: String? = null,
    @SerializedName("display_name")
    val displayName: String? = null,
    @SerializedName("avatar_url")
    val avatarUrl: String? = null,
    @SerializedName("user_type")
    val userType: UserType = UserType.REGULAR,
    @SerializedName("is_online")
    var isOnline: Boolean? = null
)

enum class UserType {
    @SerializedName("regular") REGULAR,
    @SerializedName("anonymous") ANONYMOUS,
    @SerializedName("guest") GUEST
}
```

**Create `sdk-android/src/main/java/com/chatsdk/models/Message.kt`**:

```kotlin
package com.chatsdk.models

import com.google.gson.annotations.SerializedName

data class Message(
    val id: String,
    @SerializedName("channel_id")
    val channelId: String,
    @SerializedName("user_id")
    val userId: String,
    val text: String? = null,
    val type: MessageType = MessageType.TEXT,
    var status: MessageStatus = MessageStatus.SENT,
    val attachments: List<Attachment>? = null,
    @SerializedName("created_at")
    val createdAt: String
)

enum class MessageType {
    @SerializedName("text") TEXT,
    @SerializedName("image") IMAGE,
    @SerializedName("video") VIDEO,
    @SerializedName("file") FILE
}

enum class MessageStatus {
    PENDING, SENT, DELIVERED, FAILED
}
```

---

### Task 3: API Service

**Create `sdk-android/src/main/java/com/chatsdk/api/ChatApiService.kt`**:

```kotlin
package com.chatsdk.api

import com.chatsdk.models.*
import retrofit2.http.*

interface ChatApiService {
    @POST("channels")
    suspend fun createChannel(@Body request: CreateChannelRequest): Channel
    
    @GET("channels")
    suspend fun getChannels(): List<Channel>
    
    @GET("channels/{id}")
    suspend fun getChannel(@Path("id") id: String): Channel
    
    @POST("messages")
    suspend fun sendMessage(@Body request: SendMessageRequest): Message
    
    @GET("channels/{channelId}/messages")
    suspend fun getMessages(
        @Path("channelId") channelId: String,
        @Query("limit") limit: Int = 50
    ): List<Message>
}

data class CreateChannelRequest(
    val type: String,
    val name: String? = null,
    val members: List<String>? = null
)

data class SendMessageRequest(
    val channelId: String,
    val text: String
)
```

---

### Task 4: Chat Client

**Create `sdk-android/src/main/java/com/chatsdk/ChatClient.kt`**:

```kotlin
package com.chatsdk

import com.chatsdk.api.ChatApiService
import com.chatsdk.api.CreateChannelRequest
import com.chatsdk.api.SendMessageRequest
import com.chatsdk.models.*
import io.socket.client.IO
import io.socket.client.Socket
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import okhttp3.OkHttpClient
import okhttp3.Interceptor
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory
import org.json.JSONObject

class ChatClient(
    private val apiUrl: String,
    private val wsUrl: String? = null,
    private val token: String? = null
) {
    private val apiService: ChatApiService
    private var socket: Socket? = null
    
    private val _connectionState = MutableStateFlow(ConnectionState.DISCONNECTED)
    val connectionState: StateFlow<ConnectionState> = _connectionState
    
    private val _messages = MutableStateFlow<List<Message>>(emptyList())
    val messages: StateFlow<List<Message>> = _messages
    
    init {
        val client = OkHttpClient.Builder()
            .addInterceptor(Interceptor { chain ->
                val request = chain.request().newBuilder()
                    .apply {
                        token?.let { addHeader("Authorization", "Bearer $it") }
                    }
                    .build()
                chain.proceed(request)
            })
            .build()
        
        val retrofit = Retrofit.Builder()
            .baseUrl(apiUrl)
            .client(client)
            .addConverterFactory(GsonConverterFactory.create())
            .build()
        
        apiService = retrofit.create(ChatApiService::class.java)
        
        token?.let { initializeWebSocket(it) }
    }
    
    private fun initializeWebSocket(token: String) {
        val opts = IO.Options().apply {
            extraHeaders = mapOf("Authorization" to listOf("Bearer $token"))
            reconnection = true
            reconnectionAttempts = 5
        }
        
        val url = wsUrl ?: apiUrl.replace("http", "ws")
        socket = IO.socket(url, opts)
        
        socket?.apply {
            on(Socket.EVENT_CONNECT) {
                _connectionState.value = ConnectionState.CONNECTED
            }
            
            on(Socket.EVENT_DISCONNECT) {
                _connectionState.value = ConnectionState.DISCONNECTED
            }
            
            on("message.new") { args ->
                val data = args[0] as JSONObject
                // Parse and emit message
            }
        }
    }
    
    fun connect() {
        socket?.connect()
    }
    
    fun disconnect() {
        socket?.disconnect()
    }
    
    suspend fun createChannel(type: String, name: String?, members: List<String>? = null): Channel {
        return apiService.createChannel(CreateChannelRequest(type, name, members))
    }
    
    suspend fun getChannels(): List<Channel> {
        return apiService.getChannels()
    }
    
    suspend fun sendMessage(channelId: String, text: String): Message {
        return apiService.sendMessage(SendMessageRequest(channelId, text))
    }
    
    suspend fun getMessages(channelId: String, limit: Int = 50): List<Message> {
        return apiService.getMessages(channelId, limit)
    }
    
    fun joinChannel(channelId: String) {
        socket?.emit("channel.join", JSONObject().put("channelId", channelId))
    }
}

enum class ConnectionState {
    CONNECTED, DISCONNECTED, CONNECTING
}
```

---

## Week 4: React Native SDK

### Task 1: React Native Setup

**Create `packages/sdk-react-native/package.json`**:

```json
{
  "name": "@chat-sdk/react-native",
  "version": "0.1.0",
  "main": "dist/index.js",
  "types": "dist/index.d.ts",
  "scripts": {
    "build": "tsc"
  },
  "peerDependencies": {
    "react": "^18.0.0",
    "react-native": "^0.72.0"
  },
  "dependencies": {
    "@chat-sdk/core": "workspace:*",
    "socket.io-client": "^4.6.1"
  }
}
```

---

### Task 2: React Native Client

**Create `packages/sdk-react-native/src/index.ts`**:

```typescript
import { ChatClient as CoreChatClient } from '@chat-sdk/core';
import { NativeModules, NativeEventEmitter, Platform } from 'react-native';

const { ChatSDKNative } = NativeModules;
const eventEmitter = new NativeEventEmitter(ChatSDKNative);

export class ChatClientRN extends CoreChatClient {
  async registerPushToken(token: string, provider: 'fcm' | 'apns') {
    if (Platform.OS === 'ios') {
      await ChatSDKNative.registerAPNSToken(token);
    } else {
      await ChatSDKNative.registerFCMToken(token);
    }
  }

  onPushNotification(callback: (notification: any) => void) {
    return eventEmitter.addListener('onPushNotification', callback);
  }

  async requestPermissions() {
    return ChatSDKNative.requestPushPermissions();
  }
}

export * from '@chat-sdk/core';
```

---

## Testing

```swift
// iOS Tests
class ChatClientTests: XCTestCase {
    func testSendMessage() async throws {
        let client = ChatClient(apiURL: "http://localhost:3000", token: "test-token")
        let message = try await client.sendMessage(channelId: "channel-1", text: "Hello")
        XCTAssertEqual(message.text, "Hello")
    }
}
```

```kotlin
// Android Tests
class ChatClientTest {
    @Test
    fun testSendMessage() = runBlocking {
        val client = ChatClient("http://localhost:3000", token = "test-token")
        val message = client.sendMessage("channel-1", "Hello")
        assertEquals("Hello", message.text)
    }
}
```

---

## Deliverables

- [x] iOS SDK (Swift Package)
- [x] Android SDK (Kotlin/Gradle)
- [x] React Native SDK
- [x] WebSocket support (all platforms)
- [x] Push notification integration
- [x] Offline message queue
- [x] Type-safe APIs
- [x] Unit tests
- [x] Example apps
- [x] Documentation

**Next**: [MONTH_08_ADVANCED.md](./MONTH_08_ADVANCED.md)
