# End-to-end message encryption: architecture and workflow

This document follows the encryption path implemented in this repository. In short: **the browser creates keys, establishes a separate Signal Protocol session with each other chat member, encrypts on the sender's device, and decrypts on the receiver's device. The Spring Boot backend stores and forwards ciphertext and key-exchange public data; it does not encrypt or decrypt message text.**

## A few terms

- **Identity key:** a user's long-lived Curve25519 key pair. The public half identifies the user; the private half must stay on the user's device.
- **Pre-key bundle:** public keys a user publishes so another device can start a session even when they are offline.
- **X3DH:** the Signal Protocol's initial key agreement, which uses identity and pre-keys to establish a shared session.
- **Double Ratchet:** the Signal Protocol mechanism used by the library to derive changing message keys and encrypt each message in a session.
- **IndexedDB:** browser-local database storage. It is not the backend's PostgreSQL database.
- **DTO:** a “data transfer object”—a Java/TypeScript shape used to send data between components or over HTTP.

The browser uses `@privacyresearch/libsignal-protocol-typescript` (`frontend/package.json`). Its `SessionBuilder` and `SessionCipher` implement the Signal Protocol work. The app code calls those APIs rather than implementing the message cipher itself.

## 1. Signup creates the user's device keys

1. `frontend/src/components/register/SignUp.tsx` collects the email and password. `frontend/src/redux/auth/AuthAction.ts` sends them to `/auth/signup`.
2. In `backend/src/main/java/com/nicolas/chatapp/controllers/AuthController.java`, Spring's `@RestController` maps the HTTP request to Java code. The backend verifies signup and OTP, hashes the account password with BCrypt, creates the user, and returns a JWT (a signed login token). The account password is for account login; it is also passed back through the browser route state so the browser can protect the local encryption key.
3. After OTP verification, `frontend/src/components/register/VerifyOtp.tsx` calls `KeyManagementService.initializeOnSignup()` and then uploads the public pre-key bundle.

The identity pair is generated in `frontend/src/services/KeyManagementService.ts`:

```ts
const identityKeyPair = await KeyHelper.generateIdentityKeyPair();
```

The code treats this as the Curve25519/X25519 identity pair. It stores the base64-encoded **public** key in the local key record and keeps the private key in a password-encrypted bundle. `frontend/src/services/CryptoService.ts` derives a wrapping key from the password and encrypts the bundle:

```ts
{ name: "PBKDF2", salt, iterations: 600000, hash: "SHA-256" }
// derived key:
{ name: "AES-GCM", length: 256 }
```

Each key record gets a fresh 16-byte salt and a fresh 12-byte AES-GCM IV. The record (`encryptedData`, `iv`, `salt`, public key) is saved by `frontend/src/services/KeyStorageService.ts` in the browser's `chatapp-keystore` IndexedDB database. The private key is also held in the service's memory cache while the user is signed in; it is not uploaded as part of this key record.

### Publishing pre-keys

`KeyManagementService.uploadPreKeyBundle()` initializes `SessionRatchetService`, which uses the Signal library to generate:

```ts
const signedPreKey = await KeyHelper.generateSignedPreKey(identityKeyPair, signedPreKeyId);
const otpk = await KeyHelper.generatePreKey(i + 1);
```

The bundle contains the identity **public** key, signed-pre-key public key and signature, and 100 one-time pre-key public keys. The corresponding signed and one-time **private** pre-keys remain in IndexedDB, encrypted at rest by `SessionRatchetService`; their public halves are sent to the server. `frontend/src/services/SessionRatchetService.ts` uploads the bundle to `PUT /api/users/pre-key-bundle`.

On the backend, `UserController` routes the request to `UserServiceImpl`; `PreKeyBundle` is a JPA entity (a Java object mapped to a database row), and `PreKeyBundleRepository` saves it. The configured database is PostgreSQL (`backend/src/main/resources/application.properties`). The server therefore has public identity/pre-key material and can return it to other users, but it does not have the matching private keys.

## 2. Sign-in reloads local keys and ratchet state

`SignIn.tsx` calls `loginUser()` in `AuthAction.ts`; the backend authenticates and returns a JWT. After fetching the user's profile, `SignIn.tsx` calls `KeyManagementService.loadOnSignIn(userId, password)`.

That method reads the key record from IndexedDB and uses `CryptoService.decryptPrivateKey()` to unwrap the identity private key. AES-GCM authentication fails if the password is wrong. On success, the service passes the public/private identity pair and a browser device ID to `SessionRatchetService.initialize()`. Initialization reloads locally stored ratchet sessions, signed/one-time private pre-keys, and pinned contact identities.

Ratchet session state and pre-key private material are encrypted at rest with AES-GCM. The service derives the storage key from the first 32 characters of the base64 private-key string using PBKDF2-SHA-256 (100,000 iterations) and a user-specific salt. This is separate from the password-derived wrapping of the identity key.

If there is no local key or it cannot be decrypted, the UI treats this as a new-device case: the user can restore from a recovery phrase or start with new keys. Starting fresh replaces the server's published bundle, so messages addressed to the old identity may no longer be decryptable with the new keys.

## 3. The sender composes and encrypts a message in React

`Homepage.tsx` owns the current chat, message list, and send handler. `MessagePage.tsx` is the input UI; Enter or the send button calls `onSendMessage()`. React state holds the text until send.

The handler in `Homepage.tsx` caches the sender's own plaintext locally, excludes the sender from the recipient list, and encrypts separately for every other member:

```ts
const ratchetResults = await Promise.all(
  targetIds.map(async recipientId => ({
    recipientId,
    result: await SessionRatchetService.ratchetEncrypt(plaintext, recipientId, token),
  }))
);
```

For a one-to-one chat this is one encryption; for a group it is one encryption per other member. The sender does **not** create a ratchet session with themself. Their own copy is instead kept in the encrypted local sent-message cache.

### First contact: fetch public keys and establish a session

When `ratchetEncrypt()` finds no saved session, `SessionRatchetService._lazyInitSession()`:

1. Fetches the recipient's bundle from `GET /api/users/{userId}/pre-key-bundle`.
2. Calls `POST /api/users/pre-key-bundle/consume` to atomically take one one-time pre-key, if available.
3. Passes the identity key, signed pre-key and signature, and consumed one-time pre-key to `SessionBuilder.processPreKey()`.

The server's `UserController` and `UserServiceImpl` perform those endpoints. The one-time key is removed under a database write lock, so concurrent senders do not receive the same pool entry. `SessionBuilder` uses the bundle to establish the initial X3DH session locally. The server sees the public bundle and the request to consume a pre-key, not the derived session secret.

The `SignalProtocolAddress` used by this code is `{theirUserId, 1}`. Although a random browser device ID is also stored and included in message metadata, the address currently uses the fixed device number `1`.

### Each message: Double Ratchet encryption

`SessionRatchetService.ratchetEncrypt()` calls the library's `SessionCipher.encrypt()` with UTF-8 bytes of the message:

```ts
const ciphertext = new TextEncoder().encode(plaintext);
const encrypted = await cipher.encrypt(ciphertext.buffer);
```

The returned binary body is base64-encoded so it can safely travel in JSON and fit in a PostgreSQL text column. The app builds a JSON ratchet header containing the message type, body, registration ID, and device ID. Type `3` is the first `PreKeyWhisperMessage` for a new session; type `1` is a normal later `WhisperMessage`. The library maintains and advances the session/ratchet state, which the app saves locally after encryption.

**Algorithm boundary:** the repository explicitly configures Curve25519/X25519 key pairs, PBKDF2-SHA-256 and AES-GCM for local storage/recovery, and calls the Signal library for X3DH and Double Ratchet messages. It does not configure or name the internal per-message cipher suite in its own code, so this document does not claim a specific inner message-cipher algorithm. In particular, the AES-GCM code in `CryptoService` is for key-at-rest protection, not the message body.

The ratchet function also creates a random 12-byte `iv` and returns it, but that IV is **not passed to `SessionCipher.encrypt()` and is not used by the decrypt path**. The ratchet header/body produced by the Signal library are what protect the message. The normal message POST below sends `content: null` and does not send this `iv`.

## 4. REST request saves message metadata and ciphertext headers

After encrypting, `Homepage.tsx` builds a map of recipient ID to that recipient's ciphertext/header, but the message request itself is:

```ts
const payload = {
  chatId: currentChat.id,
  content: null,
  ratchetHeader: JSON.stringify(ratchetHeaders),
  encryptionFormat: "ratchet",
};
dispatch(createMessage(payload, token, plaintext, tempMessageId));
```

`frontend/src/redux/message/MessageAction.ts` sends this JSON to `POST /api/messages/create` with the JWT in the `Authorization: Bearer ...` header. The Spring `MessageController` resolves the user from the JWT and delegates to `MessageServiceImpl`. The service checks that the message uses the ratchet format and has a header, then saves a `Message` entity through `MessageRepository`.

The important data actually persisted is therefore:

- `content`: `null` for this new send path.
- `ratchetHeader`: JSON mapping each recipient ID to their encrypted body and ratchet metadata.
- `encryptionFormat`: `"ratchet"`.
- `iv`: `null` for the HTTP request.
- sender, chat, timestamp, and read metadata.

`Message` maps these fields to the database; `MessageDTO` maps them back to the frontend. The backend validates and stores the ciphertext/header data, but has no message-decryption step.

## 5. STOMP/WebSocket sends each member their own encrypted copy

After the REST request succeeds, `MessageAction.ts` receives the saved message ID, re-keys the sender's temporary local plaintext-cache entry to that ID, and dispatches it into Redux. `Homepage.tsx` watches the Redux `newMessage` value and sends it to `/app/messages` over STOMP (using SockJS at `/ws`). STOMP is a messaging protocol layered over a WebSocket connection; here it lets the server publish to per-user topics.

The WebSocket payload includes the saved message and the `perRecipientContent` map that was kept in a React ref during sending. `RealtimeChatController.receiveMessage()` loops through chat members and publishes to `/topic/{userId}`. For each member with an entry in the map, it merges that member's own ciphertext into the outgoing payload. The sender is not in the map, so they receive the stored message shape and render their own text from the local cache.

The server forwards encrypted copies; it does not decrypt them. The separately generated `iv` may appear in this WebSocket copy, but `ratchetDecrypt()` does not read it.

## 6. The receiver fetches, selects, and decrypts

`Homepage.tsx` dispatches `getAllMessages()` when a chat opens or refreshes. `MessageAction.ts` calls `GET /api/messages/chat/{chatId}`. The `MessageController` checks the logged-in user and `MessageServiceImpl` verifies that they belong to the chat before `MessageRepository` returns the stored records.

For live delivery, the receiver subscribes to their `/topic/{userId}` topic. `Homepage.onMessageReceive()` parses the STOMP JSON and dispatches `RECEIVE_WEBSOCKET_MESSAGE`; `MessageReducer` merges it into the message list, retaining the per-recipient copy so a later REST refresh does not overwrite it.

`MessagePage.tsx` renders a `MessageCard` for each message. For another user's ratchet message, `MessageCard.tsx` calls:

```ts
SessionRatchetService.ratchetDecrypt(
  message.ratchetHeader,
  message.content || "",
  senderUserId,
  message.id.toString()
);
```

The service selects the header/ciphertext for the current recipient. In live delivery, `content` is already that recipient's copy. In message history, the database `content` is null, so `_resolveCiphertextForCurrentUser()` takes the current user's encrypted `body` from the saved header map.

The service base64-decodes the body and selects the library operation based on the header type:

```ts
if (header.type === 3) {
  decrypted = await cipher.decryptPreKeyWhisperMessage(bodyBuffer, "binary");
} else {
  decrypted = await cipher.decryptWhisperMessage(bodyBuffer, "binary");
}
```

The library uses the receiver's local identity and pre-key private keys for the initial message and the saved session state for later messages. On success, the browser decodes the resulting bytes into text and shows it in the message bubble. Updated session state is saved locally. If decryption fails, the UI displays “Cannot decrypt”; it does not ask the backend to decrypt.

For the sender's own message, `MessageCard` instead reads the locally cached plaintext. That cache is encrypted with AES-GCM using a key derived from the sender's Curve25519 private-key material. It is per-browser/device: the server does not have this plaintext cache, and another device will not automatically get the sender's own historical plaintext.

## 7. Recovery and key changes

`RecoveryService.generateMnemonic()` creates a 12-word phrase from browser-generated random entropy and a SHA-256 checksum. `VerifyOtp.tsx` shows the phrase and, after confirmation, encrypts a bundle containing the identity private **and public** key using a key derived from the phrase with PBKDF2-SHA-256 (600,000 iterations) and AES-256-GCM. It uploads only `{encryptedData, iv, salt}` to `/api/recovery/blob`. `RecoveryController`, `RecoveryServiceImpl`, `RecoveryBlob`, and `RecoveryBlobRepository` store that encrypted blob in the backend database.

For recovery, `SignIn.tsx` calls `RecoveryService.recoverAccess()`: the browser fetches the blob, decrypts it with the phrase, re-wraps the private key with the new account password, saves the key record in IndexedDB, and calls `loadOnSignIn()` to restore the ratchet. The backend stores the encrypted blob but does not know the phrase or have a decryption key.

Changing the account password also re-wraps the local private-key bundle in `KeyManagementService.changePassword()`; the caller separately changes the server-side account password. Regenerating keys is different: it creates a new identity and uploads a new pre-key bundle. The old identity's message history may no longer be decryptable unless the old key was recovered first.

## What this implementation does not show or guarantee

- **Message cipher internals:** application code delegates these to the pinned Signal library dependency; it does not specify the underlying message cipher suite itself.
- **First-contact identity verification:** remote identity keys are accepted and pinned on first use (TOFU: “trust on first use”), then a silently changed key is rejected. `getSafetyNumber()` exists, but this repository has no call site displaying a code for users to compare. Trust on first use does not independently prove who controlled the key when it was first fetched.
- **Full multi-device sync:** private keys and sessions are browser-local, and the backend keeps one current pre-key bundle per user. Device identity is not included in the `SignalProtocolAddress` (fixed device number `1`).
- **Sender history on a new device:** the sender's own plaintext is a local cache; it is not uploaded as plaintext or synchronized by the server.
- **Message IV field:** the ratchet message path creates/forwards an IV value but does not use it to encrypt or decrypt the message. AES-GCM IVs are used separately for local key, session, and cache encryption.
- **Pre-key replenishment:** a `refreshPreKeysIfNeeded()` method exists, and signup/sign-in upload a fresh bundle, but no periodic/background caller is evident in the frontend.

## Important files, in execution order

| Order | File | Role in the flow |
|---|---|---|
| 1 | `frontend/src/components/register/SignUp.tsx` | Collects signup details. |
| 2 | `frontend/src/redux/auth/AuthAction.ts` | Sends signup, OTP, and login HTTP requests; stores the returned JWT. |
| 3 | `backend/src/main/java/com/nicolas/chatapp/controllers/AuthController.java` | Verifies OTP/login, creates the account, and issues the JWT. |
| 4 | `frontend/src/components/register/VerifyOtp.tsx` | Starts E2EE setup after signup and handles the recovery phrase confirmation. |
| 5 | `frontend/src/services/KeyManagementService.ts` | Creates/loads identity keys; initializes ratchet and publishes pre-keys. |
| 6 | `frontend/src/services/CryptoService.ts` | Password-wraps/un-wraps the private identity key using PBKDF2 + AES-GCM. |
| 7 | `frontend/src/services/KeyStorageService.ts` | Persists identity, sessions, pre-keys, pinned identities, and local sent-message cache in IndexedDB. |
| 8 | `frontend/src/services/SessionRatchetService.ts` | Generates pre-keys, establishes X3DH sessions, encrypts/decrypts messages, and persists ratchet state. |
| 9 | `backend/src/main/java/com/nicolas/chatapp/controllers/UserController.java` | Receives, serves, and consumes pre-key bundle data. |
| 10 | `backend/src/main/java/com/nicolas/chatapp/service/implementation/UserServiceImpl.java` | Updates bundles and consumes one-time pre-keys transactionally. |
| 11 | `backend/src/main/java/com/nicolas/chatapp/model/PreKeyBundle.java` and `repository/PreKeyBundleRepository.java` | Map public bundle data to PostgreSQL and lock bundle rows during consumption. |
| 12 | `frontend/src/components/Homepage.tsx` and `components/messagePage/MessagePage.tsx` | Handle chat state, composition, per-member encryption, REST dispatch, and STOMP connection/subscription. |
| 13 | `frontend/src/redux/message/MessageAction.ts` | POSTs encrypted message metadata and GETs chat history. |
| 14 | `backend/src/main/java/com/nicolas/chatapp/controllers/MessageController.java` | Authenticates message REST requests and returns message DTOs. |
| 15 | `backend/src/main/java/com/nicolas/chatapp/service/implementation/MessageServiceImpl.java` | Validates and saves the ratchet message fields. |
| 16 | `backend/src/main/java/com/nicolas/chatapp/model/Message.java`, `repository/MessageRepository.java`, and `dto/response/MessageDTO.java` | Persist and serialize message records. |
| 17 | `backend/src/main/java/com/nicolas/chatapp/config/WebSocketConfig.java` and `controllers/RealtimeChatController.java` | Configure SockJS/STOMP and route per-recipient ciphertext to user topics. |
| 18 | `frontend/src/redux/message/MessageReducer.ts` | Merges live messages with fetched history without discarding the live recipient copy. |
| 19 | `frontend/src/components/messageCard/MessageCard.tsx` | Chooses own-message cache or receiver-side decryption and renders the result. |
| 20 | `frontend/src/services/RecoveryService.ts`, `backend/.../controllers/RecoveryController.java`, `service/implementation/RecoveryServiceImpl.java`, and `model/RecoveryBlob.java` | Encrypt, store, retrieve, and restore the recovery-encrypted identity key bundle. |

## End-to-end summary

1. Signup verifies the account → browser generates Curve25519 identity keys → private key is password-wrapped into IndexedDB.
2. Browser generates signed and one-time pre-keys → public bundle is stored by backend; private pre-keys stay encrypted in that browser.
3. Sender opens a chat → fetches recipient public bundle and consumes one one-time key if available → Signal library establishes X3DH session.
4. Sender encrypts separately for each other member with `SessionCipher` → ciphertext is base64-encoded and ratchet state is saved locally.
5. Backend saves message metadata/header map (new-send `content` is null) → STOMP forwards each recipient their own encrypted copy.
6. Recipient decrypts in their browser using local private pre-keys/session → plaintext appears in React; sender's own plaintext comes from the encrypted local cache.
7. Recovery phrase can restore the encrypted identity key bundle on a new device; without the old key or phrase, old messages can be unreadable.

**Flow:** `Compose → per-recipient Signal encryption → REST save of ciphertext metadata → STOMP delivery → recipient browser decrypts → display plaintext`
