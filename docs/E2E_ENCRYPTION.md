# End-to-End Encryption (E2EE) Design

**Status**: Design Specification  
**Version**: 1.0  
**Last Updated**: November 1, 2025

---

## Overview

This document describes the end-to-end encryption (E2EE) implementation for the Chat SDK, ensuring that messages can only be read by the sender and intended recipients, not by the server or any intermediaries.

**Protocol**: Signal Protocol (X3DH + Double Ratchet)  
**Algorithm**: Curve25519 (ECDH) + AES-256-GCM

---

## 1. Architecture Overview

```
User A (Sender)                                    User B (Recipient)
┌──────────────┐                                   ┌──────────────┐
│ Plaintext    │                                   │ Plaintext    │
│ Message      │                                   │ Message      │
└──────┬───────┘                                   └──────▲───────┘
       │ Encrypt with                                     │ Decrypt with
       │ Recipient's                                      │ Own Private Key
       │ Public Key                                       │
┌──────▼───────┐    Send Encrypted Message         ┌──────┴───────┐
│ Ciphertext   ├───────────────────────────────────►│ Ciphertext   │
└──────────────┘    (Server cannot read)           └──────────────┘
                         │
                  ┌──────▼──────┐
                  │   Server    │
                  │ • Stores    │
                  │   encrypted │
                  │ • Cannot    │
                  │   decrypt   │
                  └─────────────┘
```

---

## 2. Key Management

### 2.1 Key Types

| Key Type | Algorithm | Lifetime | Purpose |
|----------|-----------|----------|---------|
| **Identity Key** | Curve25519 | Permanent | User's identity |
| **Signed Pre-Key** | Curve25519 | 30 days | Initial key exchange |
| **One-Time Pre-Keys** | Curve25519 | Single-use | Perfect forward secrecy |
| **Session Keys** | AES-256-GCM | Per message | Encrypt messages |

### 2.2 Database Schema

```sql
-- User encryption keys
CREATE TABLE user_encryption_keys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  device_id VARCHAR(255) NOT NULL,
  identity_public_key TEXT NOT NULL,
  signed_prekey_id INTEGER NOT NULL,
  signed_prekey_public TEXT NOT NULL,
  signed_prekey_signature TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(app_user_id, device_id)
);

-- One-time pre-keys
CREATE TABLE one_time_prekeys (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  device_id VARCHAR(255) NOT NULL,
  prekey_id INTEGER NOT NULL,
  public_key TEXT NOT NULL,
  is_used BOOLEAN DEFAULT false,
  used_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(app_user_id, device_id, prekey_id)
);

-- Encrypted messages
CREATE TABLE encrypted_messages (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  message_id UUID NOT NULL REFERENCES messages(id) ON DELETE CASCADE,
  recipient_device_id VARCHAR(255) NOT NULL,
  ciphertext TEXT NOT NULL,
  ephemeral_public_key TEXT,
  used_prekey_id INTEGER,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(message_id, recipient_device_id)
);

-- Session state
CREATE TABLE encryption_sessions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  app_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  device_id VARCHAR(255) NOT NULL,
  peer_user_id UUID NOT NULL REFERENCES app_users(id) ON DELETE CASCADE,
  peer_device_id VARCHAR(255) NOT NULL,
  session_state TEXT NOT NULL,
  last_message_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(app_user_id, device_id, peer_user_id, peer_device_id)
);
```

---

## 3. Key Exchange Protocol (X3DH)

### 3.1 Initial Setup

```typescript
// 1. Client generates keys
const keyBundle = {
  identityKey: generateKeyPair(),
  signedPreKey: generateSignedPreKey(),
  oneTimePreKeys: generateOneTimePreKeys(100)
};

// 2. Upload public keys to server
await uploadPublicKeys({
  identityPublicKey: keyBundle.identityKey.public,
  signedPreKey: {
    id: keyBundle.signedPreKey.id,
    public: keyBundle.signedPreKey.public,
    signature: keyBundle.signedPreKey.signature
  },
  oneTimePreKeys: keyBundle.oneTimePreKeys.map(k => ({
    id: k.id,
    public: k.public
  }))
});

// 3. Store private keys securely (client-side only)
await secureStorage.store('keys', keyBundle);
```

### 3.2 Sending First Message

```typescript
// 1. Fetch recipient's public key bundle
const recipientKeys = await fetchKeyBundle(recipientUserId, recipientDeviceId);

// 2. Verify signed pre-key signature
const isValid = verifySignature(
  recipientKeys.identityPublicKey,
  recipientKeys.signedPreKey.public,
  recipientKeys.signedPreKey.signature
);

// 3. Perform X3DH key agreement
const sharedSecret = x3dhKeyAgreement({
  myIdentityKey,
  myEphemeralKey: generateEphemeralKey(),
  recipientIdentityKey: recipientKeys.identityPublicKey,
  recipientSignedPreKey: recipientKeys.signedPreKey.public,
  recipientOneTimePreKey: recipientKeys.oneTimePreKey?.public
});

// 4. Initialize session and encrypt
const session = initializeSession(sharedSecret);
const ciphertext = session.encrypt(plaintext);

// 5. Send encrypted message
await sendMessage({
  recipientUserId,
  recipientDeviceId,
  ciphertext,
  ephemeralPublicKey: myEphemeralKey.public,
  usedPreKeyId: recipientKeys.oneTimePreKey?.id
});
```

---

## 4. Double Ratchet Algorithm

### 4.1 Purpose

- **Forward Secrecy**: Past messages remain secure if keys compromised
- **Post-Compromise Security**: Future messages become secure again

### 4.2 Implementation

```typescript
interface RatchetState {
  dhSendingKey: KeyPair;
  dhReceivingKey: PublicKey;
  sendingChainKey: Buffer;
  receivingChainKey: Buffer;
  sendingCounter: number;
  receivingCounter: number;
  skippedMessageKeys: Map<number, Buffer>;
}

// Encrypt message
async function ratchetEncrypt(
  state: RatchetState,
  plaintext: string
): Promise<{ ciphertext: string; newState: RatchetState }> {
  const messageKey = deriveMessageKey(state.sendingChainKey);
  const ciphertext = aesGcmEncrypt(plaintext, messageKey);
  const newChainKey = ratchetChainKey(state.sendingChainKey);
  
  return {
    ciphertext,
    newState: {
      ...state,
      sendingChainKey: newChainKey,
      sendingCounter: state.sendingCounter + 1
    }
  };
}

// Decrypt message
async function ratchetDecrypt(
  state: RatchetState,
  ciphertext: string,
  messageNumber: number
): Promise<{ plaintext: string; newState: RatchetState }> {
  // Handle out-of-order messages
  if (state.skippedMessageKeys.has(messageNumber)) {
    const messageKey = state.skippedMessageKeys.get(messageNumber)!;
    const plaintext = aesGcmDecrypt(ciphertext, messageKey);
    state.skippedMessageKeys.delete(messageNumber);
    return { plaintext, newState: state };
  }
  
  // Derive message key
  const messageKey = deriveMessageKey(state.receivingChainKey);
  const plaintext = aesGcmDecrypt(ciphertext, messageKey);
  
  return {
    plaintext,
    newState: {
      ...state,
      receivingChainKey: ratchetChainKey(state.receivingChainKey),
      receivingCounter: messageNumber + 1
    }
  };
}
```

---

## 5. API Endpoints

### 5.1 Key Management

```typescript
// Upload public keys
POST /api/v1/encryption/keys
{
  deviceId: string;
  identityPublicKey: string;
  signedPreKey: { id: number; public: string; signature: string };
  oneTimePreKeys: Array<{ id: number; public: string }>;
}

// Fetch recipient's keys
GET /api/v1/encryption/keys/:userId/:deviceId
Response: {
  identityPublicKey: string;
  signedPreKey: { id: number; public: string; signature: string };
  oneTimePreKey?: { id: number; public: string } | null;
}

// Replenish one-time pre-keys
POST /api/v1/encryption/keys/replenish
{
  deviceId: string;
  oneTimePreKeys: Array<{ id: number; public: string }>;
}

// Rotate signed pre-key (every 30 days)
POST /api/v1/encryption/keys/rotate-signed-prekey
{
  deviceId: string;
  newSignedPreKey: { id: number; public: string; signature: string };
}
```

### 5.2 Encrypted Messaging

```typescript
// Send encrypted message
POST /api/v1/channels/:channelId/messages/encrypted
{
  recipients: Array<{
    userId: string;
    deviceId: string;
    ciphertext: string;
    ephemeralPublicKey?: string;
    usedPreKeyId?: number;
  }>;
  metadata: {
    type: 'text' | 'image' | 'file';
    timestamp: string;
  };
}

// Fetch encrypted messages
GET /api/v1/channels/:channelId/messages/encrypted?deviceId=<id>&limit=50
Response: {
  messages: Array<{
    messageId: string;
    senderId: string;
    senderDeviceId: string;
    ciphertext: string;
    ephemeralPublicKey?: string;
    usedPreKeyId?: number;
    timestamp: string;
  }>;
}
```

---

## 6. Client SDK Implementation

```typescript
// Initialize encryption
const encryptionManager = new EncryptionManager(secureStorage);
await encryptionManager.initialize(userId, deviceId);

// Send encrypted message
const plaintext = 'Hello, secret message!';
const recipients = await getChannelMembers(channelId);

const encryptedPayloads = await Promise.all(
  recipients.flatMap(member =>
    member.devices.map(device =>
      encryptionManager.encryptMessage(
        member.userId,
        device.id,
        plaintext
      )
    )
  )
);

await client.sendEncryptedMessage(channelId, {
  recipients: encryptedPayloads,
  metadata: { type: 'text', timestamp: new Date().toISOString() }
});

// Receive encrypted message
client.on('message.encrypted', async (event) => {
  const plaintext = await encryptionManager.decryptMessage(
    event.senderId,
    event.senderDeviceId,
    event.ciphertext
  );
  console.log('Decrypted:', plaintext);
});
```

---

## 7. Security Considerations

### 7.1 What E2EE Protects

✅ Message content (text, attachments)  
✅ Message edits  
✅ Reactions  
✅ Read receipts (optional)

### 7.2 What E2EE Does NOT Protect

❌ Sender/recipient identities  
❌ Message timestamps  
❌ Channel membership  
❌ Typing indicators  
❌ Online/offline status

### 7.3 Key Storage

**Client-Side:**
- Web: IndexedDB + Web Crypto API
- iOS: Keychain Services
- Android: Android Keystore
- Desktop: OS credential managers

**Server-Side:**
- Only store public keys
- Never store or log private keys
- Encrypt database at rest

### 7.4 Multi-Device Support

- Each device has its own key pair
- Messages encrypted separately per device
- Devices cannot decrypt each other's messages

---

## 8. Implementation Phases

### Phase 1: Core Encryption (Month 1)
- [ ] X3DH key exchange
- [ ] Double Ratchet algorithm
- [ ] Database schema
- [ ] Key generation/storage

### Phase 2: API Integration (Month 2)
- [ ] Key management endpoints
- [ ] Encrypted message endpoints
- [ ] Key rotation logic

### Phase 3: SDK Integration (Month 3)
- [ ] EncryptionManager class
- [ ] Secure storage adapters
- [ ] Multi-device support

### Phase 4: UI/UX (Month 4)
- [ ] Encryption indicators
- [ ] Key verification UI
- [ ] Device management

### Phase 5: Testing & Audit (Month 5)
- [ ] Unit tests
- [ ] Integration tests
- [ ] Security audit

---

## 9. Performance Impact

**Key Generation:**
- Initial setup: ~600ms
- Acceptable one-time cost

**Message Encryption:**
- Single message: ~5-10ms
- Group (10 recipients): ~50-100ms
- Negligible for real-time chat

**Storage:**
- Per user: ~7 KB + (512 bytes × peers)
- Minimal impact

**Network:**
- Additional ~32 bytes per message
- ~3% overhead

---

## 10. Compliance

### 10.1 Legal Considerations

- E2EE may be restricted in some countries
- Implement region-based feature flags
- Provide non-encrypted fallback
- Server cannot decrypt messages
- Can provide metadata (not content)

### 10.2 User Consent

- Clear disclosure of E2EE usage
- Explain limitations
- Allow opt-in/opt-out per channel
- Provide unencrypted channels for compliance

---

## References

- [Signal Protocol Specification](https://signal.org/docs/)
- [X3DH Key Agreement](https://signal.org/docs/specifications/x3dh/)
- [Double Ratchet Algorithm](https://signal.org/docs/specifications/doubleratchet/)
- [libsignal-protocol-javascript](https://github.com/signalapp/libsignal-protocol-javascript)
