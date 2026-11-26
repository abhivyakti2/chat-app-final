# End-to-end message encryption: architecture and workflow

This document follows the encryption path implemented in this repository. **End-to-End Encryption (E2EE)** [message protection where only the communicating users' devices have the keys to read its content] happens in the browser: it establishes a separate **Signal Protocol** [a protocol suite for creating secure sessions and protecting messages] session with each other chat member, encrypts on the sender's device, and decrypts on the receiver's device. The **Spring Boot backend** [the Java framework that runs this server application] stores and forwards **ciphertext** [encrypted data that should not reveal its original message] and public **key-exchange data** [public values devices use to establish shared secrets]; it does not encrypt or decrypt message text.

The browser uses `@privacyresearch/libsignal-protocol-typescript` (`frontend/package.json`), a **cryptographic library** [reusable code implementing cryptographic operations]. Its **SessionBuilder** [the library component that uses public pre-keys to create a new secure session] and **SessionCipher** [the library component that encrypts and decrypts messages in an existing session] implement the Signal Protocol work. The app calls those APIs rather than implementing the message cipher itself.

## 1. Signup creates the user's device keys

1. `frontend/src/components/register/SignUp.tsx` collects the email and password. `frontend/src/redux/auth/AuthAction.ts` sends them in an **HTTP Request** [a message sent by a client to a server] using **HTTP** [the web's request/response communication protocol] to `/auth/signup`.
2. In `backend/src/main/java/com/nicolas/chatapp/controllers/AuthController.java`, Spring's `@RestController` [a Spring component that receives web requests and returns responses] maps the HTTP request to Java code. The backend verifies signup and **OTP** [One-Time Password, a short-lived code used once to verify an action], performs **password hashing** [a one-way transformation used to check passwords, not reversible encryption] with **BCrypt** [the specific one-way password-hashing algorithm used here], creates the user, and returns a **JWT** [JSON Web Token, a signed token carrying login/authentication information]. The account password is for account login; it is also passed back through browser route state so the browser can protect the local encryption key.
3. After OTP verification, `frontend/src/components/register/VerifyOtp.tsx` calls `KeyManagementService.initializeOnSignup()` and then uploads the public pre-key bundle. **Key Management** [the app logic that creates, loads, and prepares cryptographic keys] is handled by `KeyManagementService`; **Key Storage** [the app logic that persists keys and session data locally] is handled by `KeyStorageService`.

The **Identity Key Pair** [the matched public and private keys for a user's cryptographic identity] is generated in `frontend/src/services/KeyManagementService.ts`. Its **Identity Key** [the long-lived key pair identifying a user to Signal sessions] uses **Curve25519** [a family of elliptic-curve cryptography algorithms] / **X25519** [Curve25519's key-agreement operation]. **Asymmetric Cryptography** [cryptography using a public/private key pair] lets a public key be shared while the **Private Key** [the secret half that must remain protected] stays on the device:

```ts
const identityKeyPair = await KeyHelper.generateIdentityKeyPair();
```

The code treats this as the Curve25519/X25519 identity pair. It stores the **Public Key** [the shareable half used by others to establish a session] in the local key record after **Base64 Encoding** [representing binary bytes as printable text] and keeps the private key in an **Encrypted Bundle** [a packaged set of key data encrypted together]. `frontend/src/services/CryptoService.ts` performs **Password Wrapping** [encrypting a private key using a key derived from the user's password]: it uses **Key Derivation** [turning password or key material into a cryptographic key] to make a **Wrapping Key** [the derived key that encrypts another key or bundle]:

```ts
{ name: "PBKDF2", salt, iterations: 600000, hash: "SHA-256" }
// derived key:
{ name: "AES-GCM", length: 256 }
```

**PBKDF2** [a password-based key-derivation algorithm] uses **SHA-256** [a hash function producing a fixed-size digest]; **PBKDF2-SHA-256** [PBKDF2 configured to use SHA-256] is the resulting configuration. The 600,000 **iterations** [repeated derivation work that makes password guessing more expensive] slow down guessing. **AES-GCM** [a symmetric encryption mode that also detects tampering] with key length 256 is **AES-256-GCM** [AES with a 256-bit key in GCM mode]; **AES-256** [AES using a 256-bit key] names that key size. **Symmetric Encryption** [encryption and decryption using the same secret key] protects the bundle. Each record gets a fresh 16-byte **Salt** [random input to key derivation so the same password does not always produce the same key] and 12-byte **Initialization Vector (IV)** [a per-encryption input required by the cipher]. **Encryption** [transforming readable data into protected data] produces an **Authentication Tag** [a value checked during decryption to detect a wrong key or tampering]. The record (`encryptedData`, `iv`, `salt`, public key) is saved by `frontend/src/services/KeyStorageService.ts` in the browser's `chatapp-keystore` **IndexedDB** [a database built into the browser, separate from the server database]. This is **At Rest Encryption** [protecting data while stored rather than while being transmitted]. The private key is also held in the service's **Local Key Cache** [short-lived in-memory copy used by the app while signed in]; it is not uploaded as part of this key record.

### Publishing pre-keys

`KeyManagementService.uploadPreKeyBundle()` initializes **SessionRatchetService** [the app service managing Signal sessions and their evolving ratchet state], which uses the Signal library to generate a **Signed Pre-key** [a medium-lived public key authenticated by the identity key] and **One-time Pre-key (OTPK)** [a single-use public key intended to be consumed once]:

```ts
const signedPreKey = await KeyHelper.generateSignedPreKey(identityKeyPair, signedPreKeyId);
const otpk = await KeyHelper.generatePreKey(i + 1);
```

The **Pre-key Bundle** [a set of public keys published so another device can start a session while the user is offline] contains the identity public key, signed-pre-key public key and signature, and 100 one-time pre-key public keys. The corresponding signed and one-time private pre-keys remain in IndexedDB, encrypted at rest by `SessionRatchetService`; their public halves are sent to the server. This is **Private Key Protection** [keeping secret key material inaccessible to other parties]. `frontend/src/services/SessionRatchetService.ts` uploads the bundle to `PUT /api/users/pre-key-bundle`.

On the backend, `UserController` routes the request to `UserServiceImpl`'s **Service Layer** [the part of an application that applies business rules]; `PreKeyBundle` is a **JPA Entity** [a Java object mapped to a database table row], and `PreKeyBundleRepository` is a **Repository** [the component that reads and writes database records] that saves it. The configured database is **PostgreSQL** [the server-side relational database] (`backend/src/main/resources/application.properties`). The server therefore has public identity/pre-key material and can return it to other users, but it does not have the matching private keys.

## 2. Sign-in reloads local keys and ratchet state

`SignIn.tsx` calls `loginUser()` in `AuthAction.ts`; the backend authenticates and returns a JWT. The browser sends the JWT in an **Authorization Header** [the HTTP header identifying the logged-in caller] using the **Bearer Token** [a token presented as proof of access] format. After fetching the user's profile, `SignIn.tsx` calls `KeyManagementService.loadOnSignIn(userId, password)`.

That method reads the key record from IndexedDB and uses `CryptoService.decryptPrivateKey()` for **Decryption** [turning ciphertext back into readable data] to unwrap the identity private key. AES-GCM **Authentication** [verifying that the key and data are valid and untampered] fails if the password is wrong. On success, the service passes the public/private identity pair and a browser **Device ID** [a random local identifier for this browser installation] to `SessionRatchetService.initialize()`. This browser identifier is a **Device Identity** [the app's label for a device instance, not a separately authenticated Signal identity in this implementation]. Initialization reloads locally stored **Ratchet State** [the evolving cryptographic values needed to continue the ratchet] and **Session State** [the saved state of a secure conversation, including ratchet progress], signed/one-time private pre-keys, and pinned contact identities.

Ratchet session state and pre-key private material are encrypted at rest with AES-GCM. The service derives the storage key from the first 32 characters of the base64 private-key string using PBKDF2-SHA-256 (100,000 iterations) and a user-specific salt. This is separate from the password-derived wrapping of the identity key.

If there is no local key or it cannot be decrypted, the UI treats this as a new-device case: the user can restore from a recovery phrase or start with new keys. Starting fresh replaces the server's published bundle, so messages addressed to the old identity may no longer be decryptable with the new keys.

## 3. The sender composes and encrypts a message in React

`Homepage.tsx` owns the current chat, message list, and send handler. `MessagePage.tsx` is the input UI; Enter or the send button calls `onSendMessage()`. **React State** [values React tracks to update the visible interface] holds the text until send. **Redux** [a shared frontend state store] keeps app-wide chat and message data coordinated between components.

The handler in `Homepage.tsx` caches the sender's own plaintext locally, excludes the sender from the recipient list, and encrypts separately for every other member:

```ts
const ratchetResults = await Promise.all(
  targetIds.map(async recipientId => ({
    recipientId,
    result: await SessionRatchetService.ratchetEncrypt(plaintext, recipientId, token),
  }))
);
```

This is **Per-Recipient Encryption** [separate encryption using a distinct session for each recipient]: a one-to-one chat has one encryption; a group has one per other member. The resulting **Per-Recipient Ciphertext** [the encrypted copy that only that recipient's session can decrypt] differs between recipients. The sender does **not** create a ratchet session with themself. Their own copy is instead kept in the encrypted local sent-message cache.

### First contact: fetch public keys and establish a session

When `ratchetEncrypt()` finds no saved session, `SessionRatchetService._lazyInitSession()` performs **Session Initialization** [creating the cryptographic state for a new conversation]:

1. Fetches the recipient's bundle from `GET /api/users/{userId}/pre-key-bundle`.
2. Calls `POST /api/users/pre-key-bundle/consume` for **Key Consumption** [removing one available one-time public key from the server's pool], if available.
3. Passes the identity key, signed pre-key and signature, and consumed one-time pre-key to `SessionBuilder.processPreKey()`.

The server's `UserController` and `UserServiceImpl` perform those endpoints. The one-time key is removed under a **Database Write Lock** [a database mechanism that makes competing updates wait their turn] inside a **Transactional Operation** [a group of database actions treated as one all-or-nothing unit], so concurrent senders do not receive the same pool entry. `SessionBuilder` uses the bundle for **X3DH** [the Signal Protocol's initial key agreement using identity and pre-keys]. **Key Agreement** [two devices independently deriving the same secret from their private keys and exchanged public keys] establishes the session locally; this is also **Key Exchange** [the process of using exchanged key material to start secure communication]. The resulting **Session Secret** [secret material shared by the two endpoints, never sent as a message] is used by the Signal library to derive **Session Keys** [short-lived keys used to protect individual messages]. The server sees the public bundle and request to consume a pre-key, not the derived session secret.

The library's **SignalProtocolAddress** [the Signal identifier pairing an account ID with a device number] is `{theirUserId, 1}`. Although a random browser Device ID is also stored and included in **Message Metadata** [non-message fields such as IDs, timestamps, and routing details], the address currently uses the fixed device number `1`.

### Each message: Double Ratchet encryption

**Ratchet** [a mechanism that advances secret key material as messages are sent] describes how each message gets fresh derived key material. The **Double Ratchet** [the Signal design combining a Diffie-Hellman ratchet and a symmetric-key ratchet to update keys over time] provides the ongoing message protection. `SessionRatchetService.ratchetEncrypt()` calls the library's `SessionCipher.encrypt()` after **UTF-8 Encoding** [converting text into a standard sequence of bytes]:

```ts
const ciphertext = new TextEncoder().encode(plaintext);
const encrypted = await cipher.encrypt(ciphertext.buffer);
```

The returned **Binary Data** [bytes that may contain values that are not valid readable text] is **Base64 Encoded** [converted into printable text safe to transport in text formats] so it can safely travel in **JSON** [a text format for structured data] and fit in a PostgreSQL text column. The app builds a **Ratchet Header** [metadata alongside the ciphertext that tells the receiver which Signal message format and body to process] containing message type, body, registration ID, and device ID. Type `3` is a **PreKeyWhisperMessage** [the first Signal encrypted message, which carries the data needed to establish a session] for a new session; type `1` is a **WhisperMessage** [a normal encrypted message in an established session]. The library maintains and advances the ratchet/session state, which the app saves locally after encryption.

**Algorithm boundary:** the repository explicitly configures Curve25519/X25519 key pairs, PBKDF2-SHA-256 and AES-GCM for local storage/recovery, and calls the Signal library for X3DH and Double Ratchet messages. **Public-Key Cryptography** [using related public and private keys for operations such as establishing a shared secret] is used in the session setup. The app does not configure or name the internal per-message cipher suite in its own code, so this document does not claim a specific inner message-cipher algorithm. In particular, the AES-GCM code in `CryptoService` is for key-at-rest protection, not the message body.

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

`frontend/src/redux/message/MessageAction.ts` sends this JSON to `POST /api/messages/create` as an **HTTP Request** [the client's message asking the server to perform an operation] to a **REST API** [web endpoints organized around resources and standard HTTP methods]. It includes the JWT in the `Authorization: Bearer ...` header. The `MessageController` is a Spring **REST Controller** [a Spring component exposing HTTP endpoints that return data]; it resolves the user from the JWT and delegates to `MessageServiceImpl`. This is the **Service Layer** [the backend component that applies application rules between the controller and database]. The service checks that the message uses the ratchet format and has a header, then saves a `Message` entity through `MessageRepository`.

The important data actually persisted is therefore:

- `content`: `null` for this new send path.
- `ratchetHeader`: JSON mapping each recipient ID to their encrypted body and ratchet metadata.
- `encryptionFormat`: `"ratchet"`.
- `iv`: `null` for the HTTP request.
- sender, chat, timestamp, and read metadata. These descriptive fields, rather than message text, are examples of Message Metadata.

`Message` maps these fields to the database; `MessageDTO` maps them back to the frontend. A **DTO** [Data Transfer Object, a simple data shape used to carry information between application parts] defines those request/response fields. `MessageDTO` is returned in an **HTTP Response** [the server's reply to the client's request]. The backend validates and stores the ciphertext/header data, but has no message-decryption step.

## 5. STOMP/WebSocket sends each member their own encrypted copy

After the REST request succeeds, `MessageAction.ts` receives the saved message ID, re-keys the sender's temporary local plaintext-cache entry to that ID, and dispatches it into Redux. `Homepage.tsx` watches the Redux `newMessage` value and sends it to `/app/messages` over **STOMP** [a messaging protocol for sending messages to destinations and subscribing to them] using **SockJS** [a browser library that provides WebSocket communication with fallback transports] at `/ws`. A **WebSocket** [a persistent two-way connection between browser and server] carries the STOMP **Messaging Protocol** [the rules and message formats used for exchanging application messages]; here this supports **Real-Time Messaging** [delivery to connected users without repeatedly polling the server]. The server publishes to a per-user **User Topic** [a named message destination that a particular user subscribes to].

The WebSocket payload includes the saved message and the `perRecipientContent` map that was kept in a React ref during sending. `RealtimeChatController.receiveMessage()` loops through chat members and publishes to `/topic/{userId}`. For each member with an entry in the map, it merges that member's own ciphertext into the outgoing payload. The sender is not in the map, so they receive the stored message shape and render their own text from the local cache.

The server forwards encrypted copies; it does not decrypt them. The separately generated `iv` may appear in this WebSocket copy, but `ratchetDecrypt()` does not read it.

## 6. The receiver fetches, selects, and decrypts

`Homepage.tsx` dispatches `getAllMessages()` when a chat opens or refreshes. `MessageAction.ts` calls `GET /api/messages/chat/{chatId}`. The `MessageController` checks the logged-in user and `MessageServiceImpl` verifies that they belong to the chat before `MessageRepository` returns the stored records.

For live delivery, the receiver subscribes to their `/topic/{userId}` topic. `Homepage.onMessageReceive()` parses the STOMP JSON and dispatches `RECEIVE_WEBSOCKET_MESSAGE`; `MessageReducer` (the Redux function that updates shared state in response to actions) merges it into the message list, retaining the per-recipient copy so a later REST refresh does not overwrite it.

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

The service base64-decodes the body and selects the library operation based on the header type. For decryption, the bytes are decoded back into text using UTF-8:

```ts
if (header.type === 3) {
  decrypted = await cipher.decryptPreKeyWhisperMessage(bodyBuffer, "binary");
} else {
  decrypted = await cipher.decryptWhisperMessage(bodyBuffer, "binary");
}
```

The library uses the receiver's local identity and pre-key private keys for the initial message and the saved session state for later messages. The original readable message is **Plaintext** [data before encryption or after successful decryption]. On success, the browser decodes the resulting bytes into text and shows it in the message bubble. Updated session state is saved locally. If decryption fails, the UI displays “Cannot decrypt”; it does not ask the backend to decrypt.

For the sender's own message, `MessageCard` instead reads the locally cached plaintext. That cache is encrypted with AES-GCM using a key derived from the sender's Curve25519 private-key material. It is per-browser/device: the server does not have this plaintext cache, and another device will not automatically get the sender's own historical plaintext.

## 7. Recovery and key changes

`RecoveryService.generateMnemonic()` creates a 12-word **Recovery Phrase** [words the user saves to unlock a backup of their encryption key]. It is a **Mnemonic** [a human-readable sequence of words representing recovery material] generated from browser-random **Entropy** [unpredictable random data used as the basis for keys or recovery words] and a SHA-256 **Checksum** [extra bits used to detect some typing mistakes in the phrase]. `VerifyOtp.tsx` shows the phrase and, after confirmation, encrypts a bundle containing the identity private **and public** key using a key derived from the phrase with PBKDF2-SHA-256 (600,000 iterations) and AES-256-GCM. This produces an **Encrypted Blob** [an opaque block of encrypted data stored or transferred as one object]; the backend receives only `{encryptedData, iv, salt}` at `/api/recovery/blob`. `RecoveryController`, `RecoveryServiceImpl`, `RecoveryBlob`, and `RecoveryBlobRepository` store that blob in the backend database.

For **Key Recovery** [restoring the user's original cryptographic keys from saved recovery material], `SignIn.tsx` calls `RecoveryService.recoverAccess()`: the browser fetches the blob, decrypts it with the phrase, re-wraps the private key with the new account password, saves the key record in IndexedDB, and calls `loadOnSignIn()` to restore the ratchet. The backend stores the encrypted blob but does not know the phrase or have a decryption key.

Changing the account password also re-wraps the local private-key bundle in `KeyManagementService.changePassword()`; the caller separately changes the server-side account password. Regenerating keys is different: it creates a new identity and uploads a new pre-key bundle. The old identity's message history may no longer be decryptable unless the old key was recovered first.

## What this implementation does not show or guarantee

- **Message cipher internals:** application code delegates these to the pinned Signal library dependency; it does not specify the underlying message cipher suite itself.
- **First-contact identity verification:** **First-Contact Identity Verification** [checking that a contact's first public key really belongs to that person] is not provided as an in-app check. Remote identity keys are accepted and pinned on first use under **TOFU** [Trust On First Use, accepting a key the first time and rejecting later silent changes]. **Identity Verification** [confirming a key belongs to the person it claims to represent] therefore depends on trusting the first fetched key; TOFU does not independently prove who controlled it. `getSafetyNumber()` calculates a **Safety Number** [a short fingerprint derived from users' identity public keys for comparing out of band], but this repository has no call site displaying a code for users to compare.
- **Full multi-device sync:** **Multi-Device Synchronization** [keeping encryption keys and message/session state consistent across a user's devices] is not implemented. Private keys and sessions are browser-local, and the backend keeps one current pre-key bundle per user. Device identity is not included in the `SignalProtocolAddress` (fixed device number `1`).
- **Sender history on a new device:** the sender's own plaintext is a local cache; it is not uploaded as plaintext or synchronized by the server.
- **Message IV field:** the ratchet message path creates/forwards an IV value but does not use it to encrypt or decrypt the message. AES-GCM IVs are used separately for local key, session, and cache encryption.
- **Pre-key replenishment:** **Key Replenishment** [creating and uploading replacement one-time pre-keys when the pool runs low] is available through `refreshPreKeysIfNeeded()`, and signup/sign-in upload a fresh bundle, but no periodic/background caller is evident in the frontend. The service's **Key Rotation** [replacing a key with a fresh one on a schedule or policy trigger] tracks signed-pre-key IDs for a seven-day interval and rotates when a bundle is generated; this is not a periodic background job.

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
