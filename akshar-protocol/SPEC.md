# Akshar Protocol — Formal Specification

> **अक्षर — That which does not perish. That which cannot be silenced.**

**Spec Version**: 1.0  
**Source**: Whitepaper v1.1 (May 2026)  
**Task**: S-002 — Protocol Specification Document  
**Status**: Draft — Source of Truth for all protocol implementation

---

## Table of Contents

1. [Overview](#1-overview)
2. [Terminology](#2-terminology)
3. [System Architecture](#3-system-architecture)
4. [Data Structures](#4-data-structures)
5. [Cryptographic Model](#5-cryptographic-model)
6. [Node Communication Protocol](#6-node-communication-protocol)
7. [Two-Layer Communication Model](#7-two-layer-communication-model)
8. [Propagation Algorithm](#8-propagation-algorithm)
9. [Token Mechanics](#9-token-mechanics)
10. [Proof of Humanity — Tiered Access](#10-proof-of-humanity--tiered-access)
11. [Sybil Resistance Model](#11-sybil-resistance-model)
12. [API Contracts](#12-api-contracts)
13. [Error Codes](#13-error-codes)
14. [Constants and Protocol Parameters](#14-constants-and-protocol-parameters)
15. [Out of Scope](#15-out-of-scope)

---

## 1. Overview

Akshar Protocol is an open, decentralised, peer-to-peer communication protocol. It unifies private group messaging (Layer 1) with attributed public broadcast (Layer 2) in a single cryptographic architecture. Every participant device is simultaneously a messaging client, a propagation relay, and a token-earning node.

### Core Design Invariants

These invariants MUST hold for any conforming implementation:

1. **Organizer blindness**: No protocol operator, foundation member, or relay node holds any decryption key for any message at any time.
2. **Hop-local traceability**: Propagation graph data is stored exclusively on individual devices. No global graph exists.
3. **Attribution permanence**: Every share act permanently and immutably attaches the sharer's node identity to the share record.
4. **Content-blind scoring**: The propagation algorithm and diversity scorer operate exclusively on graph topology. They never access, parse, or hash message content.
5. **Non-purchasable node score**: Node score can only be earned through demonstrated curation quality; it cannot be acquired by payment.

---

## 2. Terminology

| Term | Definition |
|---|---|
| **Node** | Any device running the Akshar protocol |
| **Client Node** | A node used for messaging, sharing, and token earning |
| **Relay Node** | An always-on node that buffers encrypted blobs for offline recipients |
| **NodeID** | The SHA-256 fingerprint of a node's Ed25519 public key |
| **Group** | A private, encrypted messaging context; membership is explicit |
| **Share** | The deliberate act of broadcasting a message from a group to the sharer's public circle |
| **Circle** | The set of nodes that follow a given node — recipients of that node's Layer 2 shares |
| **Feed** | The ordered stream of Share events visible to a node from all nodes it follows |
| **Cluster** | A topologically dense subgraph of the social graph; nodes within a cluster have high connectivity to each other and low connectivity to nodes outside |
| **Diversity Score** | A per-message float measuring the fraction of propagation that crosses cluster boundaries |
| **Velocity** | The current propagation rate of a message, measured in shares per time window |
| **Node Score** | A per-node float [0.0, 1.0] representing the historical curation quality of that node |
| **$RSN** | The native protocol token; usage-reward model |
| **ZK Proof** | A zero-knowledge proof used for Proof-of-Origin claims |
| **HopRecord** | A local-only record of a single propagation hop; never transmitted |
| **PoH** | Proof of Humanity — the tiered trust system |
| **Larva / Drone / Colony** | PoH tiers 1 / 2 / 3 |

---

## 3. System Architecture

### 3.1 Three-Layer Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     APPLICATION LAYER                        │
│            (Reference App — iOS, Android, Web)               │
└──────────────────────────┬──────────────────────────────────┘
                           │ protocol API
┌──────────────────────────▼──────────────────────────────────┐
│                      PROTOCOL LAYER                          │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────────┐  │
│  │ Network     │  │ Privacy      │  │ Economic           │  │
│  │ Layer       │  │ Layer        │  │ Layer              │  │
│  │             │  │              │  │                    │  │
│  │ P2P mesh    │  │ E2E crypto   │  │ Token ledger       │  │
│  │ Relay nodes │  │ ZK proofs    │  │ Reward engine      │  │
│  │ Graph router│  │ Hop records  │  │ Node score         │  │
│  └─────────────┘  └──────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Node Component Model

```
┌──────────────────── Client Node ────────────────────────────┐
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────┐  │
│  │ Message      │   │ Crypto       │   │ Local Store     │  │
│  │ Handler      │◄──│ Engine       │   │                 │  │
│  │              │   │              │   │ - Messages      │  │
│  │ - send       │   │ - keygen     │   │ - HopRecords    │  │
│  │ - receive    │   │ - encrypt    │   │ - ZK sigs       │  │
│  │ - reply      │   │ - decrypt    │   │ - NodeScore     │  │
│  │ - react      │   │ - sign       │   │ - Groups        │  │
│  └──────┬───────┘   └──────────────┘   └─────────────────┘  │
│         │                                                    │
│  ┌──────▼───────┐   ┌──────────────┐   ┌─────────────────┐  │
│  │ Propagation  │   │ Token        │   │ PoH Engine      │  │
│  │ Engine       │   │ Wallet       │   │                 │  │
│  │              │   │              │   │ - tier state    │  │
│  │ - share()    │   │ - balance    │   │ - puzzle eval   │  │
│  │ - score()    │   │ - claim()    │   │ - vouch()       │  │
│  │ - diversity()│   │ - zk_sign()  │   │ - upgrade()     │  │
│  └──────────────┘   └──────────────┘   └─────────────────┘  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────── Relay Node ─────────────────────────────┐
│                                                              │
│  ┌──────────────┐   ┌──────────────┐   ┌─────────────────┐  │
│  │ Relay        │   │ Blob Buffer  │   │ Uptime Oracle   │  │
│  │ Service      │   │              │   │                 │  │
│  │              │   │ encrypted    │   │ - heartbeat     │  │
│  │ - receive    │   │ blobs only   │   │ - report()      │  │
│  │ - buffer     │   │ no keys      │   │ - claim()       │  │
│  │ - forward    │   │              │   │                 │  │
│  └──────────────┘   └──────────────┘   └─────────────────┘  │
│                                                              │
│  INVARIANT: Relay nodes hold zero decryption keys.           │
│  A seized relay node reveals no message content.             │
└──────────────────────────────────────────────────────────────┘
```

---

## 4. Data Structures

All sizes are in bytes unless noted. All integers are big-endian. All timestamps are Unix epoch milliseconds (int64).

### 4.1 NodeID

```
NodeID := SHA-256( Ed25519PublicKey )
         = [32]byte
```

### 4.2 Message

The fundamental unit of content. Exists in one of two layer contexts.

```
Message {
    message_id        [16]byte        // UUIDv4
    version           uint8           // protocol version; currently 1
    layer             uint8           // 0x01 = GROUP, 0x02 = SHARE
    group_id          [16]byte        // UUIDv4; zero-value if layer = SHARE
    author_node_id    [32]byte        // NodeID of original author
    timestamp_utc     int64           // Unix ms
    reply_to          [16]byte        // parent message_id; zero-value if root
    content_hash      [32]byte        // SHA-256 of encrypted_payload
    encrypted_payload []byte          // E2E encrypted; opaque to all non-members
    zk_origin_token   []byte          // optional; ZKProof bytes (see §5.5); nil until claimed
    signature         [64]byte        // Ed25519 sig over all preceding fields
}
```

**Invariant**: `content_hash == SHA-256(encrypted_payload)`. Any node receiving a message MUST verify this before storing or forwarding.

**Invariant**: `signature` MUST verify against `author_node_id`'s public key over the canonical serialisation of all preceding fields.

### 4.3 ShareRecord

Created at Layer 2 share time. Immutable after creation.

```
ShareRecord {
    share_id              [16]byte    // UUIDv4
    version               uint8       // protocol version; currently 1
    source_message_id     [16]byte    // Message.message_id being shared
    sharer_node_id        [32]byte    // NodeID of the node performing the share
    attribution_chain     [][32]byte  // ordered [Originator, ..., Sharer]
    share_timestamp       int64       // Unix ms
    originating_group_id  [16]byte    // group where source message first appeared
    signature             [64]byte    // Ed25519 sig by sharer_node_id
}
```

**Invariant**: The last element of `attribution_chain` MUST equal `sharer_node_id`.  
**Invariant**: `signature` MUST verify against `sharer_node_id`'s public key.

### 4.4 FeedEvent

The unit delivered to a node's Layer 2 feed.

```
FeedEvent {
    event_id          [16]byte        // UUIDv4
    share_record      ShareRecord
    diversity_score   DiversityScore  // snapshot at delivery time
    velocity          float32         // shares/min at delivery time
    delivered_at      int64           // Unix ms; local node clock
}
```

### 4.5 HopRecord

**Local-only. Never transmitted. Never persisted outside the device.**

```
HopRecord {
    hop_id            [16]byte        // UUIDv4; local only
    message_id        [16]byte        // Message.message_id
    received_from     [32]byte        // NodeID
    forwarded_to      [][32]byte      // NodeIDs
    hop_timestamp     int64           // Unix ms
}
```

**Invariant**: HopRecords MUST NOT be included in any outbound network message.  
**Invariant**: HopRecords MUST NOT be uploaded to any remote store.

### 4.6 DiversityScore

Computed and maintained by each node's local Propagation Engine. The on-chain ledger stores only the snapshot attached to a TokenClaim.

```
DiversityScore {
    message_id            [16]byte    // Message.message_id
    cluster_spread        float32     // fraction of hops crossing cluster boundaries [0.0, 1.0]
    unique_clusters       uint32      // count of distinct topological clusters reached
    total_shares          uint64      // cumulative share count at scoring time
    velocity              float32     // shares/min; exponential moving average
    velocity_modifier     float32     // [-1.0, 1.0]; applied to base velocity
    scored_at             int64       // Unix ms
    reeval_pending        bool        // true if cluster→diverse transition detected
}
```

### 4.7 NodeIdentity

```
NodeIdentity {
    node_id               [32]byte    // NodeID = SHA-256(public_key)
    public_key            [32]byte    // Ed25519 public key
    tier                  uint8       // 0x01=LARVA, 0x02=DRONE, 0x03=COLONY
    tier_attained_at      int64       // Unix ms
    node_score            float32     // [0.0, 1.0]
    score_updated_at      int64       // Unix ms
    stake                 uint64      // $RSN micro-units; non-zero for COLONY only
}
```

### 4.8 Group

```
Group {
    group_id              [16]byte    // UUIDv4
    version               uint8
    admin_node_ids        [][32]byte  // NodeIDs with admin rights
    member_node_ids       [][32]byte  // all members including admins
    symmetric_key_ref     [32]byte    // key identifier; actual key held by members only
    is_sealed             bool        // if true, sharing permanently disabled
    created_at            int64       // Unix ms
    updated_at            int64       // Unix ms
    group_signature       [64]byte    // Ed25519 sig by founding admin
}
```

### 4.9 TokenClaim

```
TokenClaim {
    claim_id              [16]byte    // UUIDv4
    claimant_node_id      [32]byte    // NodeID
    message_id            [16]byte    // Message.message_id
    role                  uint8       // see Role enum below
    zk_proof              []byte      // ZKProof bytes
    diversity_snapshot    DiversityScore
    reward_amount         uint64      // $RSN micro-units
    chain_tx_id           [32]byte    // on-chain transaction hash
    claimed_at            int64       // Unix ms
}

// Role enum
ROLE_ORIGINATOR     = 0x01
ROLE_EARLY_SHARER   = 0x02
ROLE_LATE_SHARER    = 0x03
ROLE_RELAY          = 0x04
ROLE_CURATOR        = 0x05
```

### 4.10 HumanityProof

```
HumanityProof {
    proof_id              [16]byte    // UUIDv4
    node_id               [32]byte    // NodeID being verified
    tier_target           uint8       // tier being applied for
    proof_type            uint8       // 0x01=POW, 0x02=PUZZLE, 0x03=VOUCH
    proof_data            []byte      // type-specific payload
    attested_by           [][32]byte  // NodeIDs of evaluators (PUZZLE/VOUCH)
    created_at            int64       // Unix ms
    expires_at            int64       // Unix ms; -1 = no expiry
}
```

---

## 5. Cryptographic Model

### 5.1 Key Types

| Purpose | Algorithm | Key Size |
|---|---|---|
| Node identity signing | Ed25519 | 32-byte private, 32-byte public |
| Group message encryption | AES-256-GCM | 32-byte symmetric key |
| Content hash | SHA-256 | 32-byte digest |
| ZK proof system | zk-SNARK (Groth16) | circuit-dependent |
| Device PoW | SHA-256 hashcash | variable difficulty |

### 5.2 Key Generation

```pseudocode
function generate_node_keypair() -> (private_key, public_key, node_id):
    (private_key, public_key) = Ed25519.keygen()
    node_id = SHA-256(public_key)
    return (private_key, public_key, node_id)

function generate_group_key() -> symmetric_key:
    return CSPRNG.bytes(32)   // 256-bit AES key
```

### 5.3 Message Signing

```pseudocode
function sign_message(msg: Message, private_key: [32]byte) -> Message:
    payload = canonical_serialise({
        msg.message_id, msg.version, msg.layer, msg.group_id,
        msg.author_node_id, msg.timestamp_utc, msg.reply_to,
        msg.content_hash, msg.encrypted_payload, msg.zk_origin_token
    })
    msg.signature = Ed25519.sign(private_key, payload)
    return msg

function verify_message(msg: Message, public_key: [32]byte) -> bool:
    payload = canonical_serialise({...})  // same fields as above
    return Ed25519.verify(public_key, payload, msg.signature)
        and msg.content_hash == SHA-256(msg.encrypted_payload)
```

### 5.4 Group Encryption

```pseudocode
function encrypt_group_message(plaintext: []byte, group_key: [32]byte) -> (ciphertext: []byte, nonce: [12]byte):
    nonce = CSPRNG.bytes(12)
    ciphertext = AES-256-GCM.encrypt(key=group_key, nonce=nonce, plaintext=plaintext)
    return (nonce || ciphertext)   // prepend nonce to ciphertext

function decrypt_group_message(payload: []byte, group_key: [32]byte) -> plaintext: []byte:
    nonce = payload[0:12]
    ciphertext = payload[12:]
    return AES-256-GCM.decrypt(key=group_key, nonce=nonce, ciphertext=ciphertext)
```

### 5.5 ZK Proof of Origin

The ZK circuit proves: *"I held the Ed25519 private key K whose public key fingerprint is NodeID N, and I signed the message with content_hash H at time T"* — without revealing K, N's real-world identity, T's exact value, or any other message the node has authored. Note: the content hash covers the encrypted payload; the ZK proof attests to the signing key, not the encryption key (group key), which is separate.

```pseudocode
// At message creation time (local only, never transmitted)
function generate_origin_witness(private_key, content_hash, timestamp) -> witness:
    witness = {
        private_key_commitment: Pedersen.commit(private_key),
        content_hash: content_hash,
        timestamp: timestamp,
        randomness: CSPRNG.bytes(32)
    }
    store witness locally only
    return witness

// At claim time (published on-chain)
function generate_zk_proof(witness, public_inputs: {content_hash, timestamp}) -> ZKProof:
    proof = Groth16.prove(
        circuit   = ORIGIN_CLAIM_CIRCUIT,
        witness   = witness,
        pub_input = public_inputs
    )
    return proof

// Verification (on-chain)
function verify_zk_proof(proof: ZKProof, content_hash: [32]byte, timestamp: int64) -> bool:
    return Groth16.verify(
        circuit   = ORIGIN_CLAIM_CIRCUIT,
        proof     = proof,
        pub_input = {content_hash, timestamp}
    )
```

---

## 6. Node Communication Protocol

### 6.1 Transport

The protocol runs over a peer-to-peer transport layer (e.g., libp2p). All connections are mutually authenticated using each node's Ed25519 identity key. All wire messages are encrypted at the transport layer in addition to application-layer E2E encryption.

### 6.2 Wire Message Envelope

```
WireMessage {
    protocol_version  uint8           // currently 0x01
    message_type      uint8           // see MessageType enum
    payload_length    uint32          // bytes
    payload           []byte          // serialised inner message
    checksum          [4]byte         // CRC-32 of payload
}

// MessageType enum
MSG_GROUP_MESSAGE       = 0x01    // Message (layer=GROUP)
MSG_SHARE_EVENT         = 0x02    // ShareRecord
MSG_FEED_DELIVERY       = 0x03    // FeedEvent
MSG_RELAY_BLOB          = 0x04    // encrypted blob for relay buffering
MSG_UPTIME_HEARTBEAT    = 0x05    // relay node uptime report
MSG_TOKEN_CLAIM         = 0x06    // TokenClaim submitted to ledger
MSG_PEER_HANDSHAKE      = 0x10    // node identity exchange
MSG_PEER_DISCONNECT     = 0x11    // graceful disconnect
MSG_TIER_UPGRADE_REQ    = 0x20    // PoH tier upgrade request
MSG_VOUCH_REQUEST       = 0x21    // Colony vouch request
MSG_VOUCH_RESPONSE      = 0x22    // Colony vouch approval/rejection
```

### 6.3 Handshake Protocol

```pseudocode
// Initiator → Responder
function initiate_handshake(local_identity: NodeIdentity) -> HandshakeInit:
    return {
        protocol_version : CURRENT_PROTOCOL_VERSION,
        node_id          : local_identity.node_id,
        public_key       : local_identity.public_key,
        tier             : local_identity.tier,
        timestamp        : now_utc_ms(),
        nonce            : CSPRNG.bytes(32),
        signature        : Ed25519.sign(private_key, {node_id, timestamp, nonce})
    }

// Responder validates and responds
function handle_handshake(init: HandshakeInit) -> (HandshakeResponse, error):
    if not Ed25519.verify(init.public_key, {init.node_id, init.timestamp, init.nonce}, init.signature):
        return (nil, ERR_INVALID_HANDSHAKE_SIG)
    if init.node_id != SHA-256(init.public_key):
        return (nil, ERR_NODEID_MISMATCH)
    if abs(init.timestamp - now_utc_ms()) > MAX_CLOCK_SKEW_MS:
        return (nil, ERR_CLOCK_SKEW)
    local = get_local_identity()
    response = HandshakeResponse{
        protocol_version : CURRENT_PROTOCOL_VERSION,
        node_id          : local.node_id,
        public_key       : local.public_key,
        tier             : local.tier,
        timestamp        : now_utc_ms(),
        nonce            : init.nonce,        // echo initiator nonce to bind response
        signature        : Ed25519.sign(local.private_key, {local.node_id, now_utc_ms(), init.nonce})
    }
    return (response, nil)
```

### 6.4 Message Routing

```pseudocode
function route_message(msg: Message, local_node: NodeIdentity, social_graph: Graph):
    // Verify before forwarding
    if not verify_message(msg, lookup_public_key(msg.author_node_id)):
        discard(msg)
        return

    // Record hop locally
    hop = HopRecord{
        hop_id        : new_uuid(),
        message_id    : msg.message_id,
        received_from : sender_node_id,
        forwarded_to  : [],
        hop_timestamp : now_utc_ms()
    }

    if msg.layer == GROUP:
        targets = group_members(msg.group_id) - {local_node.node_id}
    else:  // SHARE
        targets = get_circle(msg.author_node_id)  // followers

    for target in targets:
        if target is online:
            send_direct(target, msg)
        else:
            relay_buffer(target, msg)   // via relay node
        hop.forwarded_to.append(target)

    store_local(hop)   // never transmitted

function relay_buffer(target_node_id: NodeID, msg: Message):
    // Relay node receives and stores only the encrypted blob
    // It has no key material and cannot decrypt
    blob = RelayBlob{
        target_node_id : target_node_id,
        payload        : msg.encrypted_payload,    // opaque bytes
        ttl            : now_utc_ms() + RELAY_TTL_MS,
        blob_id        : new_uuid()
    }
    send_to_relay(nearest_relay_node(), blob)
```

---

## 7. Two-Layer Communication Model

### 7.1 Layer 1 — Group Operations

```pseudocode
function create_group(admin: NodeIdentity, initial_members: []NodeID) -> Group:
    group_key = generate_group_key()
    group = Group{
        group_id        : new_uuid(),
        version         : CURRENT_PROTOCOL_VERSION,
        admin_node_ids  : [admin.node_id],
        member_node_ids : [admin.node_id] + initial_members,
        symmetric_key_ref: SHA-256(group_key),
        is_sealed       : false,
        created_at      : now_utc_ms(),
        updated_at      : now_utc_ms()
    }
    group.group_signature = Ed25519.sign(admin.private_key, canonical_serialise(group))
    distribute_group_key(group_key, group.member_node_ids)  // E2E encrypted to each member
    return group

function send_group_message(author: NodeIdentity, group: Group, plaintext: []byte, reply_to: UUID) -> Message:
    if group.is_sealed and author.node_id not in group.admin_node_ids:
        return error(ERR_GROUP_SEALED)
    group_key = get_group_key(group.group_id)   // from local store
    encrypted_payload = encrypt_group_message(plaintext, group_key)
    msg = Message{
        message_id    : new_uuid(),
        version       : CURRENT_PROTOCOL_VERSION,
        layer         : GROUP,
        group_id      : group.group_id,
        author_node_id: author.node_id,
        timestamp_utc : now_utc_ms(),
        reply_to      : reply_to,           // nil for root messages
        content_hash  : SHA-256(encrypted_payload),
        encrypted_payload: encrypted_payload,
        zk_origin_token: nil
    }
    msg = sign_message(msg, author.private_key)

    // Store ZK origin witness locally (never transmitted)
    witness = generate_origin_witness(author.private_key, msg.content_hash, msg.timestamp_utc)
    store_local_witness(msg.message_id, witness)

    route_message(msg, author, social_graph)
    return msg

function seal_group(admin: NodeIdentity, group: Group) -> Group:
    if admin.node_id not in group.admin_node_ids:
        return error(ERR_NOT_ADMIN)
    group.is_sealed = true
    group.updated_at = now_utc_ms()
    broadcast_group_update(group)
    return group
```

### 7.2 Layer 2 — Share Operations

```pseudocode
function share_message(sharer: NodeIdentity, source_msg: Message, group: Group) -> (ShareRecord, error):
    if sharer.tier < DRONE:
        return (nil, ERR_TIER_INSUFFICIENT)   // LARVA cannot share
    if group.is_sealed:
        return (nil, ERR_GROUP_SEALED)
    if sharer.node_id not in group.member_node_ids:
        return (nil, ERR_NOT_GROUP_MEMBER)

    // Build attribution chain: extend the source message's chain
    // First share of a message: chain starts with the original author
    prev_chain = get_attribution_chain(source_msg)   // [author_node_id] if first share
    attribution_chain = prev_chain + [sharer.node_id]

    share_record = ShareRecord{
        share_id             : new_uuid(),
        version              : CURRENT_PROTOCOL_VERSION,
        source_message_id    : source_msg.message_id,
        sharer_node_id       : sharer.node_id,
        attribution_chain    : attribution_chain,
        share_timestamp      : now_utc_ms(),
        originating_group_id : group.group_id,
    }
    share_record.signature = Ed25519.sign(sharer.private_key, canonical_serialise(share_record))

    // Broadcast the share to sharer's circle
    circle = get_circle(sharer.node_id)
    feed_event = FeedEvent{
        event_id       : new_uuid(),
        share_record   : share_record,
        diversity_score: get_or_init_diversity_score(source_msg.message_id),
        velocity       : compute_current_velocity(source_msg.message_id),
        delivered_at   : now_utc_ms()
    }
    for recipient in circle:
        deliver_feed_event(recipient, feed_event)

    // Notify group members that a share occurred (attributed)
    notify_group_of_share(group, sharer.node_id, source_msg.message_id, share_record.share_id)

    // Update diversity score with this new share hop
    update_diversity_score(source_msg.message_id, sharer.node_id)

    return (share_record, nil)
```

### 7.3 Feed Interactions

```pseudocode
function like_feed_item(node: NodeIdentity, event_id: UUID):
    record_reaction(event_id, node.node_id, REACTION_LIKE)
    apply_velocity_delta(event_id, +LIKE_VELOCITY_DELTA)    // see §14: LIKE_VELOCITY_DELTA

function dislike_feed_item(node: NodeIdentity, event_id: UUID):
    record_reaction(event_id, node.node_id, REACTION_DISLIKE)
    apply_velocity_delta(event_id, -DISLIKE_VELOCITY_DELTA) // see §14: DISLIKE_VELOCITY_DELTA

function share_from_feed(node: NodeIdentity, event_id: UUID) -> (ShareRecord, error):
    feed_event = get_feed_event(event_id)
    source_msg = get_message(feed_event.share_record.source_message_id)
    // Re-shares from the feed extend the attribution chain
    return share_message(node, source_msg, virtual_feed_group())
```

---

## 8. Propagation Algorithm

### 8.1 Diversity Score Initialisation

```pseudocode
function init_diversity_score(message_id: UUID) -> DiversityScore:
    return DiversityScore{
        message_id       : message_id,
        cluster_spread   : 0.0,
        unique_clusters  : 0,
        total_shares     : 0,
        velocity         : 0.0,
        velocity_modifier: 0.0,
        scored_at        : now_utc_ms(),
        reeval_pending   : false
    }
```

### 8.2 Topological Diversity Update

```pseudocode
function update_diversity_score(message_id: UUID, new_sharer_node_id: NodeID):
    score = get_diversity_score(message_id)  // or init if absent
    sharer_cluster = get_cluster(new_sharer_node_id)

    previous_clusters = get_clusters_reached(message_id)
    is_new_cluster = sharer_cluster not in previous_clusters

    score.total_shares += 1

    if is_new_cluster:
        score.unique_clusters += 1
        add_cluster_to_reached(message_id, sharer_cluster)

    // Recompute cross-cluster fraction
    cross_cluster_hops = count_cross_cluster_hops(message_id)
    score.cluster_spread = cross_cluster_hops / score.total_shares

    // Velocity: exponential moving average over a 10-minute window
    time_delta_min = (now_utc_ms() - score.scored_at) / 60_000.0
    instant_velocity = 1.0 / max(time_delta_min, 0.001)
    score.velocity = EMA_ALPHA * instant_velocity + (1 - EMA_ALPHA) * score.velocity

    // Velocity modifier based on diversity
    score.velocity_modifier = compute_velocity_modifier(score.cluster_spread, score.unique_clusters)

    // Flag for re-evaluation if transitioning from clustered to diverse
    was_clustered = score.cluster_spread < DIVERSITY_THRESHOLD_LOW
    is_now_diverse = score.cluster_spread >= DIVERSITY_THRESHOLD_HIGH
    if was_clustered and is_now_diverse:
        score.reeval_pending = true

    score.scored_at = now_utc_ms()
    store_diversity_score(score)

function compute_velocity_modifier(cluster_spread: float32, unique_clusters: uint32) -> float32:
    // Pure topology signal — no content access
    if cluster_spread >= DIVERSITY_THRESHOLD_HIGH and unique_clusters >= MIN_CLUSTERS_FOR_BOOST:
        // Diverse spread: positive boost, capped at MAX_VELOCITY_BOOST
        boost = min(cluster_spread * DIVERSITY_BOOST_FACTOR, MAX_VELOCITY_BOOST)
        return boost
    elif cluster_spread < DIVERSITY_THRESHOLD_LOW:
        // Clustered spread: proportional reduction
        insularity = 1.0 - cluster_spread
        reduction = insularity * INSULARITY_REDUCTION_FACTOR
        return -min(reduction, MAX_VELOCITY_REDUCTION)
    else:
        return 0.0   // neutral band
```

### 8.3 Effective Velocity Computation

```pseudocode
function compute_effective_velocity(message_id: UUID, sharer_node_id: NodeID) -> float32:
    score = get_diversity_score(message_id)
    node_score = get_node_score(sharer_node_id)
    base_velocity = BASE_VELOCITY * node_score
    effective = base_velocity * (1.0 + score.velocity_modifier)
    return clamp(effective, MIN_VELOCITY, MAX_VELOCITY)
```

### 8.4 Node Score Update

```pseudocode
function update_node_score(node_id: NodeID, share_id: UUID):
    // Called after diversity outcomes are known for a message the node shared
    current_score = get_node_score(node_id)
    share_outcome = get_diversity_score(get_message_id(share_id))

    // Quality signal: did this share contribute to diverse spread?
    if share_outcome.cluster_spread >= DIVERSITY_THRESHOLD_HIGH:
        quality_delta = SCORE_QUALITY_INCREASE * (1.0 - current_score)  // diminishing returns
    else:
        quality_delta = -SCORE_QUALITY_DECAY

    // Cap the increase rate
    quality_delta = clamp(quality_delta, -MAX_SCORE_DELTA, MAX_SCORE_DELTA)

    new_score = clamp(current_score + quality_delta, 0.0, 1.0)

    // Temporal decay: score decays toward baseline if node is inactive
    time_since_last_share = now_utc_ms() - get_last_share_time(node_id)
    if time_since_last_share > SCORE_DECAY_WINDOW_MS:
        decay = SCORE_DECAY_RATE * (time_since_last_share / SCORE_DECAY_WINDOW_MS)
        new_score = max(new_score - decay, SCORE_FLOOR)

    store_node_score(node_id, new_score, now_utc_ms())
```

### 8.5 Re-evaluation Pass

```pseudocode
function run_reeval_pass():
    // Periodic background task; runs every REEVAL_INTERVAL_MS
    pending = get_messages_with_reeval_pending()
    for message_id in pending:
        score = get_diversity_score(message_id)
        if score.cluster_spread >= DIVERSITY_THRESHOLD_HIGH:
            // Retroactively re-score early sharers upward
            early_sharers = get_sharers_before_threshold(message_id)
            for sharer_id in early_sharers:
                update_node_score(sharer_id, get_share_id(sharer_id, message_id))
            score.reeval_pending = false
            store_diversity_score(score)
```

---

## 9. Token Mechanics

### 9.1 Reward Pool

```pseudocode
// Reward pool sources
RewardPool {
    subscription_revenue  : uint64   // $RSN from paid subscriptions
    protocol_inflation    : uint64   // scheduled inflation per epoch
    epoch_start           : int64    // Unix ms
    epoch_duration_ms     : int64    // = EPOCH_DURATION_MS
}
```

### 9.2 Originator Claim

```pseudocode
function claim_originator_reward(node: NodeIdentity, message_id: UUID) -> (TokenClaim, error):
    msg = get_message(message_id)
    if msg.author_node_id != node.node_id:
        return (nil, ERR_NOT_AUTHOR)

    diversity = get_diversity_score(message_id)
    if diversity.cluster_spread < ORIGINATOR_DIVERSITY_THRESHOLD:
        return (nil, ERR_DIVERSITY_THRESHOLD_NOT_MET)

    witness = get_local_witness(message_id)
    if witness == nil:
        return (nil, ERR_NO_ORIGIN_WITNESS)

    zk_proof = generate_zk_proof(witness, {msg.content_hash, msg.timestamp_utc})

    reward = compute_originator_reward(diversity)

    claim = TokenClaim{
        claim_id           : new_uuid(),
        claimant_node_id   : node.node_id,
        message_id         : message_id,
        role               : ROLE_ORIGINATOR,
        zk_proof           : zk_proof,
        diversity_snapshot : diversity,
        reward_amount      : reward,
        claimed_at         : now_utc_ms()
    }
    // Submit to on-chain ledger for verification and payout
    chain_tx_id = submit_claim_to_ledger(claim)
    claim.chain_tx_id = chain_tx_id
    return (claim, nil)

function compute_originator_reward(diversity: DiversityScore) -> uint64:
    base = ORIGINATOR_BASE_REWARD
    diversity_multiplier = 1.0 + (diversity.cluster_spread * ORIGINATOR_DIVERSITY_MULTIPLIER)
    unique_cluster_bonus = diversity.unique_clusters * UNIQUE_CLUSTER_BONUS
    return uint64(base * diversity_multiplier + unique_cluster_bonus)
```

### 9.3 Sharer Reward Attribution

```pseudocode
function attribute_sharer_rewards(message_id: UUID):
    // Called when diversity threshold is crossed
    diversity = get_diversity_score(message_id)
    all_shares = get_all_shares(message_id)  // ordered by share_timestamp
    threshold_time = get_threshold_crossing_time(message_id)

    for share in all_shares:
        if share.share_timestamp < threshold_time:
            role = ROLE_EARLY_SHARER
            reward = compute_early_sharer_reward(share, diversity)
        else:
            role = ROLE_LATE_SHARER
            reward = compute_late_sharer_reward(share, diversity)

        // Apply curator multiplier if applicable
        curator_multiplier = get_curator_multiplier(share.sharer_node_id)
        reward = uint64(reward * curator_multiplier)

        enqueue_reward(share.sharer_node_id, message_id, role, reward)

function compute_early_sharer_reward(share: ShareRecord, diversity: DiversityScore) -> uint64:
    // Reward decreases as more shares precede this one (time-decay curve)
    position = get_share_position(share.share_id)   // 1-indexed
    base = EARLY_SHARER_BASE_REWARD
    decay = pow(EARLY_SHARER_DECAY_BASE, position - 1)
    return uint64(base * decay)

function compute_late_sharer_reward(share: ShareRecord, diversity: DiversityScore) -> uint64:
    return LATE_SHARER_BASE_REWARD   // flat, minimal
```

### 9.4 Relay Node Reward

```pseudocode
function compute_relay_reward(relay_node_id: NodeID, epoch: uint64) -> uint64:
    uptime_fraction = get_uptime_fraction(relay_node_id, epoch)
    return uint64(RELAY_BASE_REWARD_PER_EPOCH * uptime_fraction)

// Uptime oracle reports heartbeats; on-chain contract verifies and pays
function relay_heartbeat(relay_node: NodeIdentity):
    heartbeat = {
        node_id   : relay_node.node_id,
        timestamp : now_utc_ms(),
        signature : Ed25519.sign(relay_node.private_key, {relay_node.node_id, now_utc_ms()})
    }
    submit_heartbeat(heartbeat)
```

### 9.5 Curator Multiplier

```pseudocode
function get_curator_multiplier(node_id: NodeID) -> float32:
    node_score = get_node_score(node_id)
    // Multiplier compounds with node score but is capped
    raw = 1.0 + (node_score * CURATOR_MULTIPLIER_FACTOR)
    return min(raw, MAX_CURATOR_MULTIPLIER)
```

### 9.6 Subscription Velocity

```pseudocode
function apply_subscription_boost(node: NodeIdentity, message_id: UUID) -> error:
    if not has_active_subscription(node.node_id):
        return ERR_NO_SUBSCRIPTION

    diversity = get_diversity_score(message_id)
    // Boost is bounded: cannot exceed organic velocity cap
    current_velocity = diversity.velocity
    boost = min(SUBSCRIPTION_VELOCITY_BOOST, MAX_VELOCITY - current_velocity)

    // Boost decays to organic baseline if no organic spread occurs within window
    apply_timed_velocity_boost(message_id, boost, SUBSCRIPTION_BOOST_DECAY_MS)
    tag_message_as_promoted(message_id)   // labelled in feed
    return nil
```

---

## 10. Proof of Humanity — Tiered Access

### 10.1 Tier Requirements

```
Tier 1 — LARVA:
  - Complete device PoW: SHA-256 hashcash with difficulty targeting ~60 minutes on commodity hardware
  - Provide SIM uniqueness hash: H = SHA-256(SIM_ICCID || DEVICE_FINGERPRINT || NONCE)
  - Capabilities: read and reply in groups only

Tier 2 — DRONE:
  - Wait 14 days since LARVA tier attainment
  - Pass Humanity Puzzle with majority approval (≥2 of 3 evaluators)
  - Burn TOKEN_BURN_DRONE $RSN from wallet
  - Establish 3 verified human connections (mutual follows with Drone+ nodes)
  - Capabilities: full Layer 2 sharing, basic token earning

Tier 3 — COLONY:
  - 90 days since DRONE tier attainment
  - Achieve cluster_spread >= DIVERSITY_THRESHOLD_HIGH on ≥3 distinct messages
  - Receive staked vouches from ≥2 existing COLONY nodes
  - Commit stake of ≥ COLONY_STAKE_MINIMUM $RSN
  - Capabilities: relay node operation, full token earning, on-chain governance
```

### 10.2 Device Proof of Work

```pseudocode
function compute_device_pow(device_fingerprint: []byte) -> PowResult:
    target = compute_pow_target(POW_DIFFICULTY_BITS)   // leading zeros in hash
    nonce = 0
    while true:
        candidate = SHA-256(device_fingerprint || uint64_to_bytes(nonce))
        if candidate < target:
            return PowResult{
                fingerprint : device_fingerprint,
                nonce       : nonce,
                hash        : candidate,
                completed_at: now_utc_ms()
            }
        nonce += 1

function verify_device_pow(result: PowResult) -> bool:
    target = compute_pow_target(POW_DIFFICULTY_BITS)
    hash = SHA-256(result.fingerprint || uint64_to_bytes(result.nonce))
    return hash == result.hash and hash < target
```

### 10.3 Humanity Puzzle

```pseudocode
function generate_humanity_puzzle(candidate_node_id: NodeID) -> Puzzle:
    // Select 5 recent high-diversity messages to present
    messages = sample_high_diversity_messages(n=5, min_cluster_spread=DIVERSITY_THRESHOLD_HIGH)
    return Puzzle{
        puzzle_id    : new_uuid(),
        candidate    : candidate_node_id,
        messages     : messages,   // content visible to candidate only
        created_at   : now_utc_ms(),
        expires_at   : now_utc_ms() + PUZZLE_EXPIRY_MS
    }

function submit_puzzle_response(candidate: NodeIdentity, puzzle_id: UUID, response_text: []byte) -> error:
    puzzle = get_puzzle(puzzle_id)
    if now_utc_ms() > puzzle.expires_at:
        return ERR_PUZZLE_EXPIRED
    // Encrypt response; distribute to 3 random Drone+ evaluators
    evaluators = sample_evaluators(n=3, min_tier=DRONE)
    for evaluator in evaluators:
        encrypted_response = encrypt_for_evaluator(response_text, evaluator.public_key)
        send_evaluation_request(evaluator.node_id, {puzzle_id, encrypted_response})
    return nil

function submit_puzzle_evaluation(evaluator: NodeIdentity, puzzle_id: UUID, approved: bool):
    record_evaluation(puzzle_id, evaluator.node_id, approved)
    evaluations = get_evaluations(puzzle_id)
    if len(evaluations) >= 3:
        approvals = count(e.approved for e in evaluations)
        if approvals >= 2:   // majority
            grant_tier_upgrade(get_candidate(puzzle_id), DRONE)

function grant_tier_upgrade(node_id: NodeID, new_tier: uint8):
    identity = get_node_identity(node_id)
    if new_tier == DRONE:
        verify_prerequisites_drone(identity)
    elif new_tier == COLONY:
        verify_prerequisites_colony(identity)
    identity.tier = new_tier
    identity.tier_attained_at = now_utc_ms()
    broadcast_tier_update(identity)
```

### 10.4 Colony Vouch

```pseudocode
function request_colony_vouch(applicant: NodeIdentity, voucher_node_id: NodeID) -> error:
    voucher = get_node_identity(voucher_node_id)
    if voucher.tier != COLONY:
        return ERR_VOUCHER_NOT_COLONY
    send_vouch_request(voucher_node_id, {
        applicant_node_id : applicant.node_id,
        request_timestamp : now_utc_ms()
    })
    return nil

function approve_colony_vouch(voucher: NodeIdentity, applicant_node_id: NodeID, stake_amount: uint64) -> error:
    if voucher.tier != COLONY:
        return ERR_VOUCHER_NOT_COLONY
    if stake_amount < COLONY_VOUCH_MIN_STAKE:
        return ERR_INSUFFICIENT_STAKE
    lock_stake(voucher.node_id, stake_amount, applicant_node_id)
    record_vouch(applicant_node_id, voucher.node_id, stake_amount, now_utc_ms())
    check_colony_prerequisites(applicant_node_id)
    return nil
```

---

## 11. Sybil Resistance Model

### 11.1 Bot Containment Invariant

A bot network is by definition a set of synthetic nodes with high intra-cluster connectivity and low or zero organic cross-cluster connectivity. Therefore:

- Bot-amplified messages will exhibit `cluster_spread < DIVERSITY_THRESHOLD_LOW`.
- The velocity modifier for such messages will be negative.
- Bot messages cannot escape the bot cluster without genuine human shares from outside, which requires those humans to put their real node identity behind the share.

**Formal statement**: Let B ⊂ V be the set of bot nodes in the social graph G = (V, E). Let m be a message authored and propagated exclusively within B. Then `diversity_score(m).cluster_spread ≈ 0`, and `compute_velocity_modifier(cluster_spread, unique_clusters) < 0`.

### 11.2 Cost-per-Bot Analysis

| Tier | Marginal Cost per Additional Bot Account |
|---|---|
| LARVA | ~60 min real compute time + unique SIM |
| DRONE | 14 day wait + token burn + 3 human connections + pass human-evaluated puzzle |
| COLONY | 90 day wait + ≥3 organic diverse messages + 2 Colony stakes + own stake |

At COLONY tier the cost is unbounded by capital requirements and time. At scale, the expected cost makes bot farming economically irrational.

---

## 12. API Contracts

### 12.1 Node API (Local RPC)

All function signatures below represent the local RPC surface between the application layer and the protocol engine. Wire encoding is implementation-defined (recommended: Protocol Buffers or MessagePack).

```
// Group operations
create_group(admin_node_id, member_node_ids)          -> (Group, error)
join_group(node_id, group_id, invite_token)           -> (Group, error)
send_message(node_id, group_id, plaintext, reply_to)  -> (Message, error)
seal_group(admin_node_id, group_id)                   -> (Group, error)
get_group_messages(node_id, group_id, since_ts, limit)-> ([]Message, error)

// Share operations
share_message(node_id, message_id, group_id)          -> (ShareRecord, error)
like_feed_event(node_id, event_id)                    -> error
dislike_feed_event(node_id, event_id)                 -> error
share_from_feed(node_id, event_id)                    -> (ShareRecord, error)
get_feed(node_id, since_ts, limit)                    -> ([]FeedEvent, error)

// Token operations
claim_originator_reward(node_id, message_id)          -> (TokenClaim, error)
get_token_balance(node_id)                            -> (uint64, error)
get_node_score(node_id)                               -> (float32, error)
get_share_history(node_id, since_ts, limit)           -> ([]ShareRecord, error)

// PoH operations
start_tier_upgrade(node_id, target_tier)              -> (UpgradeSession, error)
submit_pow_result(node_id, pow_result)                -> error
get_puzzle(node_id)                                   -> (Puzzle, error)
submit_puzzle_response(node_id, puzzle_id, response)  -> error
request_vouch(node_id, voucher_node_id)               -> error

// Node operations
get_node_identity(node_id)                            -> (NodeIdentity, error)
get_diversity_score(message_id)                       -> (DiversityScore, error)
connect_peer(node_id, peer_multiaddr)                 -> error
disconnect_peer(node_id, peer_node_id)                -> error
```

### 12.2 On-Chain Contract Interface

```
// Token ledger
submit_token_claim(claim: TokenClaim)                 -> (tx_hash, error)
get_reward_balance(node_id)                           -> uint64
withdraw_rewards(node_id, amount, destination)        -> (tx_hash, error)

// Relay uptime oracle
submit_uptime_heartbeat(heartbeat: UptimeHeartbeat)   -> error
get_uptime_fraction(relay_node_id, epoch)             -> float32
claim_relay_reward(relay_node_id, epoch)              -> (tx_hash, error)

// Governance
submit_proposal(proposer_node_id, proposal: bytes)    -> (proposal_id, error)
cast_vote(voter_node_id, proposal_id, vote: bool)     -> error
get_voting_weight(node_id)                            -> float32   // f($RSN, node_score)
execute_proposal(proposal_id)                         -> (tx_hash, error)
```

---

## 13. Error Codes

```
ERR_INVALID_HANDSHAKE_SIG       = 1001   // signature verification failed on handshake
ERR_NODEID_MISMATCH             = 1002   // node_id != SHA-256(public_key)
ERR_CLOCK_SKEW                  = 1003   // timestamp delta exceeds MAX_CLOCK_SKEW_MS
ERR_CONTENT_HASH_MISMATCH       = 1004   // content_hash != SHA-256(encrypted_payload)
ERR_INVALID_MESSAGE_SIG         = 1005   // Ed25519 signature verification failed
ERR_GROUP_SEALED                = 2001   // share attempted on sealed group
ERR_NOT_GROUP_MEMBER            = 2002   // operation requires group membership
ERR_NOT_ADMIN                   = 2003   // operation requires admin rights
ERR_TIER_INSUFFICIENT           = 3001   // operation requires higher PoH tier
ERR_PUZZLE_EXPIRED              = 3002   // humanity puzzle submission after expiry
ERR_VOUCHER_NOT_COLONY          = 3003   // vouch requested from non-Colony node
ERR_INSUFFICIENT_STAKE          = 3004   // stake amount below minimum
ERR_DIVERSITY_THRESHOLD_NOT_MET = 4001   // originator claim before diversity threshold
ERR_NO_ORIGIN_WITNESS           = 4002   // local ZK witness missing for message
ERR_NOT_AUTHOR                  = 4003   // claim attempted by non-author
ERR_NO_SUBSCRIPTION             = 4004   // subscription boost requested without subscription
ERR_INVALID_ZK_PROOF            = 4005   // ZK proof verification failed on-chain
```

---

## 14. Constants and Protocol Parameters

These values represent initial protocol parameters. All are subject to on-chain governance revision.

```
// Protocol
CURRENT_PROTOCOL_VERSION        = 1
MAX_CLOCK_SKEW_MS               = 30_000          // 30 seconds

// Relay
RELAY_TTL_MS                    = 604_800_000     // 7 days
MAX_BLOB_PAYLOAD_BYTES          = 10_485_760      // 10 MB max payload per blob

// Diversity scoring
DIVERSITY_THRESHOLD_LOW         = 0.15            // below = clustered
DIVERSITY_THRESHOLD_HIGH        = 0.40            // above = diverse
MIN_CLUSTERS_FOR_BOOST          = 3               // minimum unique clusters for velocity boost
DIVERSITY_BOOST_FACTOR          = 0.6             // diversity → velocity boost multiplier
INSULARITY_REDUCTION_FACTOR     = 0.4             // insularity → velocity reduction multiplier
MAX_VELOCITY_BOOST              = 0.5             // cap on positive velocity modifier
MAX_VELOCITY_REDUCTION          = 0.4             // cap on negative velocity modifier
EMA_ALPHA                       = 0.3             // exponential moving average factor
BASE_VELOCITY                   = 1.0             // baseline shares/min for score=1.0 node
MIN_VELOCITY                    = 0.1
MAX_VELOCITY                    = 10.0
REEVAL_INTERVAL_MS              = 300_000         // 5 minutes

// Node score
SCORE_QUALITY_INCREASE          = 0.05
SCORE_QUALITY_DECAY             = 0.02
MAX_SCORE_DELTA                 = 0.1             // per-update cap
SCORE_DECAY_WINDOW_MS           = 2_592_000_000   // 30 days
SCORE_DECAY_RATE                = 0.01            // per decay window
SCORE_FLOOR                     = 0.01            // minimum non-zero score

// Token rewards (in $RSN micro-units; 1 $RSN = 1_000_000 micro-units)
ORIGINATOR_BASE_REWARD          = 500_000         // 0.5 $RSN
ORIGINATOR_DIVERSITY_MULTIPLIER = 2.0
ORIGINATOR_DIVERSITY_THRESHOLD  = 0.40            // matches DIVERSITY_THRESHOLD_HIGH
UNIQUE_CLUSTER_BONUS            = 50_000          // 0.05 $RSN per unique cluster
EARLY_SHARER_BASE_REWARD        = 100_000         // 0.1 $RSN (position 1)
EARLY_SHARER_DECAY_BASE         = 0.85            // geometric decay per position
LATE_SHARER_BASE_REWARD         = 10_000          // 0.01 $RSN flat
RELAY_BASE_REWARD_PER_EPOCH     = 200_000         // 0.2 $RSN per epoch at 100% uptime
CURATOR_MULTIPLIER_FACTOR       = 1.5
MAX_CURATOR_MULTIPLIER          = 3.0
EPOCH_DURATION_MS               = 86_400_000      // 24 hours

// Subscription
SUBSCRIPTION_VELOCITY_BOOST     = 2.0             // additional velocity units
SUBSCRIPTION_BOOST_DECAY_MS     = 3_600_000       // 1 hour decay window

// Feed reactions
LIKE_VELOCITY_DELTA             = 0.1             // velocity added per like
DISLIKE_VELOCITY_DELTA          = 0.15            // velocity subtracted per dislike

// PoH
POW_DIFFICULTY_BITS             = 24              // leading zero bits; targets ~60 min on mobile
TOKEN_BURN_DRONE                = 1_000_000       // 1 $RSN
COLONY_STAKE_MINIMUM            = 10_000_000      // 10 $RSN
COLONY_VOUCH_MIN_STAKE          = 5_000_000       // 5 $RSN per voucher
PUZZLE_EXPIRY_MS                = 86_400_000      // 24 hours
```

---

## 15. Out of Scope

The following are explicitly deferred and MUST NOT be assumed from this document:

- Wire encoding format (Protobuf schema, packet framing, field numbering)
- Transport layer implementation (specific libp2p configuration, NAT traversal)
- Cryptographic primitive version pinning beyond algorithm names
- Smart contract ABI, deployment target, and chain selection
- Mobile and web client UI/UX
- Relay node hardware reference specifications
- Cross-post bridge to Twitter/X
- Initial governance parameter values beyond those listed in §14
- Multi-language / internationalisation support
- Video and long-form content (Protocol v2)
- SDK public API surface (separate specification)

---

*Akshar Protocol SPEC.md v1.0 — S-002*  
*Source of truth for all protocol implementation. Supersedes Whitepaper v1.1 wherever they conflict.*
