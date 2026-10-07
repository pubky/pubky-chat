# One chat for Pubky: unification plan

**For:** engineers on Rooms, Hypercolor, Paykit, Pubky core, Nexus, Ring/Passport, pubky.app and the Shop.

**Related documents:**

- SSO: [Pubky SSO design](https://github.com/pubky/pubky-marketplace/blob/master/docs/sso/pubky-sso-design.md) and [SSO proposal for the team](https://github.com/pubky/pubky-marketplace/blob/master/docs/sso/sso-proposal-for-team.md).
- Shop messaging context:
  - [Shop team brief](https://github.com/pubky/pubky-marketplace/blob/master/docs/launch/shop-team-brief.md);
  - [Paykit team brief](https://github.com/pubky/pubky-marketplace/blob/master/docs/spec-feedback/paykit-team-brief.md);
  - the Shop's [messaging research and implementation notes](https://github.com/BitcoinErrorLog/pubky-app/blob/release/shop-v0.6.8/docs/ecommerce/messaging/README.md).
- Chat spec today: [pubky-chat `spec/`](https://github.com/pubky/pubky-chat/tree/main/spec).

Sizes follow the SSO plan:

- **S:** one component, no wire change.
- **M:** several modules or a new UI surface; any wire change is additive.
- **L:** a protocol change with new verification rules and coordination across repos.

## Answer

- **Crypto: MLS (RFC 9420)** for every private conversation, through OpenMLS. It sits behind the library's `ChatTransport` seam, and the `pubky-chat` kinds-v2 vocabulary is carried inside it. Paykit Encrypted Links remain Paykit's payment channel.
- **Identity:** a chat device key counts only when the user's grant names it (the `att` claim).
- **Recovery:** the user's pubky backup. The signer delivers scoped keys for `/priv/chat/v1/` (K6), requested as `/priv/chat/v1/:rwe`. The library derives wrapping keys from them for the random archive and inbox keys stored at stable paths there. A recovery code is the fallback.
- **Discovery and scale use four layers that work together:**

  | Layer | What it does |
  |---|---|
  | **L1, storage** | Homeservers are the source of truth. Each author owns their ciphertext under `/pub/chat/v1/`, at rotating, padded paths |
  | **L2, first-contact mailbox** | Anyone can drop a sealed knock to a profile, gated against spam. Native target: an append-only inbox on the recipient's homeserver. Bridge until then: knocks on the sender's own homeserver under a blinded tag, served by an indexer |
  | **L3, scale** | A change-feed index records, per conversation tag, the cursor of the last change. Clients fetch only from homeservers that changed |
  | **L4, resilience** | Direct crawling of contacts whenever the index is down or untrusted |

  The index is an optimisation, never a dependency.
- **No OSS messaging stack** (Matrix, XMTP, Nostr NIP-17). We reuse OpenMLS and their proven patterns (§3.9).
- **Severin's audit** found 12 root causes in the Shop's messaging. Each maps to the design element or phase item that removes it (§2). His four programs, his message lifecycle (accepted → queued → transmitted → acknowledged) and his product definitions are adopted. All six of his open decisions are settled (§3.7).
- **The Shop now (Phase 0):**
  - current messaging is frozen and labeled **beta**, for buyer–seller and mutual-follow chats;
  - key pinning and at-rest encryption already shipped in v0.6.45;
  - the Shop team fixes the two P0s only: the receive cap defers instead of consuming, and sign-out keeps the already-encrypted history.
- **Hub:** [pubky/pubky-chat](https://github.com/pubky/pubky-chat).

## 1. Prior work and decisions

### 1.1 Sources

| Source | What it is | Status |
|---|---|---|
| [pubky-rooms](https://github.com/secondl1ght/pubky-rooms) `1c5e16b` (Matt) | Public live rooms in Phoenix LiveView. Author-owned files under `/pub/pubky-rooms/`. Grant held by the server. Tag/Nexus discovery | Feature-complete, on staging |
| [hypercolor](https://github.com/BitcoinErrorLog/hypercolor) `6713b2f` | React Native messenger on Encrypted Links | Android 1.1.0. [#7](https://github.com/BitcoinErrorLog/hypercolor/pull/7) (SDK-managed links) open |
| [hypercolor-web](https://github.com/BitcoinErrorLog/hypercolor-web) `9534b79` | Web client, plus [ADRs 0001–0004](https://github.com/BitcoinErrorLog/hypercolor-web/tree/main/docs/adr) and the [graph review](https://github.com/BitcoinErrorLog/hypercolor-web/blob/main/docs/graph-utilisation-review.md) | ADRs are Proposed. [#7](https://github.com/BitcoinErrorLog/hypercolor-web/pull/7) (stock Ring auth) open |
| hypercolor-web branch `feat/homeserver-migration-proof`, `docs/architecture-comparison.md` | Hypercolor compared with Signal and Keet | Branch only |
| [pubky-chat](https://github.com/pubky/pubky-chat) `fcdb094` | `kinds-v2` (21 `chat.*` kinds), capabilities, admission, store schema, vectors, CI | Spec only |
| pubky-chat package specification, rev2 (21 Sep 2026) | `@pubky/chat-*` packages and public API; program decisions D1–D9 | Passed independent review and security audit. Not published |
| SDK-managed links migration design, rev2 (21–23 Sep) | Owner decisions 1–9 on Encrypted-Link recovery | Passed the security audit |
| First-contact root-cause analysis (Sep 2026) | A stranger's first message is never seen | Not published |
| [pubky-app-specs#142](https://github.com/pubky/pubky-app-specs/pull/142) | Social specs v1: app-neutral `{pub,priv}/social/v1/` paths and an owner-only `/priv/` tier | Open RFC |
| **Severin's audit (2 Oct 2026)** | Read-only source audit of the Shop's messaging: 77 findings in 12 root causes. Estimates 60%+ of the messaging code needs rewriting. Unknown senders are undiscoverable, and sync takes 30+ minutes for accounts with many follows | Shared in team Slack |
| **Orlando's question (2 Oct 2026)** | Should we adopt an existing p2p messaging library now and replace it later? | §3.9 |
| **Andrei's scoped keys (2 Oct 2026)** | Pubky SDK derivation of stable keys scoped to grant paths (with Sev), delivered beside the grant | SDK draft in [pubky-homeserver#668](https://github.com/pubky/pubky-homeserver/pull/668) (6 Oct). §3.2, K6 |
| **Ben's Paykit answers (2 Oct 2026)** | Shared Paykit state per identity is by design; no WASM package, storage interface or custom-message API; signed Noise-key proof in [paykit-rs#169](https://github.com/pubky/paykit-rs/pull/169) | §4, §5 |

### 1.2 Decisions already made, and what this plan does with them

**Adopt** means the decision is kept as is. **Adapt** keeps its intent with a stated change. **Supersede** replaces it, with the argument given.

| # | Decision (source) | Verdict | Reason |
|---|---|---|---|
| P1 | One `chat.*` vocabulary; listings are `chat.context.v0`, offers `chat.proposal.*` (D2) | **Adopt** | Transport-independent; carried inside MLS |
| P2 | Repo `pubky/pubky-chat`, MIT (D1) | **Adopt** | §6 |
| P3 | Package suite `chat-core`, `chat-store`, `chat-transport-paykit`, `chat-attachments`, `chat-payments`, `chat-react`, `chat-backup`, `native/` (package spec §1) | **Adopt**, plus `chat-transport-mls` and `chat-index-client` | The layering already isolates transport |
| P4 | Never TypeScript crypto; web runs the Rust state machine through WASM | **Adopt** | MLS engine is Rust (OpenMLS), built for WASM and UniFFI |
| P5 | Production primitives only: no Sealed Blob v2, UKD/AppCert, Molt, drop relay, legacy `pubky-noise` | **Adopt** | `att` is a claim in the official grant. The knock seal is HPKE (RFC 9180). The L2 bridge is author-owned files plus an index, not a relay that holds messages |
| P6 | SDK-managed Encrypted Links; zero remote deletes; re-delivery by `event_id`; never rotate the receiver key to repair one link (D4, migration decisions 1–9) | **Adopt for Hypercolor's transition** | The Shop is frozen instead (§4) |
| P7 | One receiver per app plus multi-receiver discovery (D3) | **Supersede** | Every app installation is an MLS device in the same group |
| P8 | Admission policies `hypercolor-wot`, `shop`, `open`; follows never auto-accept (D5) | **Adopt** | Admission decides Inbox versus Requests (§3.7) |
| P9 | Request rows render only the pubky, the arrival time and local facts; profiles resolve only when the row is opened (ADR 0004 §7) | **Adopt** | |
| P10 | Outbound status `Queued` / `Sent` / `Failed`; no "delivered" or "read" until a receipt protocol ships (UX contract §B.3) | **Adapt** | Severin's lifecycle is the receipt protocol. `Delivered` and `Read` render once acknowledgements ship in Phase 2 (§3.7) |
| P11 | Foreground drain; an opt-in, content-free push waker later (D7) | **Adopt** | Phase 4 |
| P12 | Random 32-byte recovery code, shown once (D6) | **Adapt** | The primary recovery path is the user's pubky backup: the signer-derived subtree seed for `/priv/chat/v1/` wraps the archive key and the inbox key (§3.2). The recovery code stays as an optional fallback until Ring, Bitkit and Passport all deliver scoped keys (K6) |
| P13 | Content-addressed attachments: `blob_id`, AAD = `blob_id`, re-seed (ADR 0001 §4) | **Adopt** | |
| P14 | DMs on Encrypted Links; groups as group-as-pubky with epoch keys; MLS "wrong first increment" (ADR 0001) | **Supersede** | §3.1 |
| P15 | No sequencer (ADR 0001) | **Adapt** | A committer orders commits only; members fork by re-creating the group (§3.5) |
| P16 | Public rooms are posts, tags and feeds (ADR 0002) | **Adapt** | Tags for discovery; messages stay Rooms' room files (§3.6) |
| P17 | First contact through a sealed-pointer drop relay; exit through `Action::Append` (ADR 0004) | **Adapt** | It becomes L2. The native target is the append-only inbox. The bridge stores knocks on the *sender's* homeserver under blinded tags, so no third party holds them |
| P18 | Sender-side publication with graph discovery, rejected because Nexus drops unknown objects and a cleartext `inbox_kid` publishes the sender's contact graph (ADR 0004) | **Supersede** | The bridge's index is a separate service that follows event streams, so no specs `Resource` variant is needed. Knocks carry no recipient identifier: the bucket is shared by many recipients, and only the recipient can verify a tag (§3.4) |
| P19 | Hypercolor has no real users; clean cutover (owner rule, 23 Sep). Kinds-v2 accepts no inbound aliases | **Adopt for Hypercolor** | The Shop keeps history (§4) |
| P20 | Public channels, GIF search, contacts, TrustEngine and AuthKeepalive stay in the host | **Adopt** | |
| P21 | Stock Ring `pubkyauth` (hypercolor#4, hypercolor-web#7) | **Adopt** | |
| P22 | Deferred: Double Ratchet, groups over 50 (dossier) | **Supersede** | MLS covers both |
| P23 | App-neutral bare-word paths; owner-only `/priv/`; mutes at `/priv/social/v1/mutes/` (social specs v1) | **Adopt** | Hence `/pub/chat/v1/`. Chat honors social-specs mutes |
| P24 | Severin's four programs, lifecycle and product definitions | **Adopt** | §2, §3.7, §5 |

**John's direction, 16 Sep:** "should we have a different thing than paykit for dedicated chat needs?" Yes: a chat transport built for chat, with Paykit kept for payments.

## 2. Severin's audit: every root cause removed

| # | Root cause | Removed by | Phase item |
|---|---|---|---|
| 1 | Delivery depends on open browser runtimes; Noise XX needs both sides to take turns | MLS is asynchronous: the sender writes a Welcome and messages to its own homeserver without the recipient, and the recipient reads them whenever it next runs (L1). Send is persisted before anything else happens (§3.7) | C3, E1 |
| 2 | Receiving depends on discovering the sender first | L2 knocks (bridge B1, native H8), with L3 for known conversations | B1, H8 |
| 3 | Bounded sync has no fairness; skipped work is lost on reload | L3 answers "what changed since my cursor". Cursors are durable per tag. L4 crawling round-robins with a durable per-peer cursor | I1, C2 |
| 4 | Durable acceptance happens too late; no recipient acknowledgement | Lifecycle accepted → queued → transmitted → acknowledged. *Accepted* is the first, local, durable write, before any policy, session or crypto work. *Acknowledged* is an automatic `chat.receipt.v0` `delivered` from a recipient device | C2, C3 |
| 5 | The receive cap consumes excess messages | **Phase 0 P0-1** in the Shop. In the library, the checkpoint advances only past messages that were persisted (kinds-v2 rule) | P0-1, C2 |
| 6 | Local storage is disposable; sign-out clears everything | **Phase 0 P0-2** in the Shop. In the library, sign-out locks the owner-scoped store but never deletes it. The encrypted archive restores history anywhere, unwrapped through the signer-derived seed or, as a fallback, the recovery code | P0-2, D1 |
| 7 | Devices share one advertised receiver marker | Every app installation is its own grant-attested device record; MLS fans out to every leaf; the self group syncs devices | C3, D1 |
| 8 | Crypto safety has no complete recovery path; stuck states and held locks | Every MLS state has a defined exit: rejoin by proposal or external commit, and committer handoff. Locks are leases with expiry. There is no "unknown" terminal state | C3 |
| 9 | Cross-tab coordination covers crypto, not the outbox | One lease per app covers MLS state **and** the outbox. Rows are claimed with a revision. Cancel and send are transitions of the same row. Ties order by `(sent_at, event_id)` | C2 |
| 10 | The UI copies snapshots instead of reading reactive state | `chat-react` hooks read live queries on `chat-store`. Drafts are their own per-conversation rows, and send completion never touches them | C2 |
| 11 | Read, unread, mute, block and request are defined inconsistently | One definition each (§3.7), in spec v3 and enforced by vectors | C1 |
| 12 | The product surface is incomplete | Kinds-v2 already specifies edit, delete, receipts, attachments, groups, pins and mentions. Bodies go to 16 KiB. `chat-store` adds pagination, timestamps and local search | C2, E1, U1 |

**Severin's four programs, mapped to phases:**

| Program | Causes | Phase |
|---|---|---|
| Reliable delivery | 1–5 | Phases 0 and 2 |
| Persistence and recovery | 6–8 | Phases 0, 2 and 3 |
| State consistency | 9–11 | Phase 1, built into the library |
| Chat usability | 12 | Phases 2 and 4 |

**His sync finding (30+ minutes at scale)** comes from probing every follow's homeserver. L3 makes cost grow with new messages, not with contacts.

**His 60%+ rewrite estimate** stands, and it is the reason the current Shop messaging is frozen rather than repaired.

## 3. Design

### 3.1 Crypto: MLS

| ADR 0001's reason against MLS | Status now |
|---|---|
| Doesn't revoke homeserver sessions | Obsolete. Grants are checked for revocation on every request (SSO §1) |
| Doesn't wipe old plaintext | True of every protocol |
| Large; not a Pubky primitive | OpenMLS is a maintained Rust implementation, shipped to web and mobile by Wire's [core-crypto](https://github.com/wireapp/core-crypto) and by XMTP |
| 50 pairwise links can carry an epoch key | They give no post-compromise security, no multi-device and no offline first message |

**Ciphersuite:** `MLS_128_DHKEMX25519_CHACHA20POLY1305_SHA256_Ed25519` (0x0003). A post-quantum hybrid suite is added when one is standardized.

**Before launch:** an independent protocol review and a security audit (X1).

### 3.2 Key hierarchy

```
pubky identity key (Ring / Bitkit / Passport; never in an app)
 ├─ grant (identity, or SSO agent under H1)
 │   cnf = app PoP key; caps include /pub/chat/v1/:rw and /priv/chat/v1/:rwe
 │   att = [{ purpose: "pubky-chat/device/v1", key: <device Ed25519 key> }]   ← K5
 │    └─ device signature key = MLS credential (one per app installation)
 │        ├─ KeyPackages
 │        └─ MLS epochs → message keys
 └─ scoped key seed S(/priv/chat/v1/) (signer-derived; delivered beside the grant; kept inside the SDK)   ← K6
     └─ file key W = K(/priv/chat/v1/keys/wrap) (what the SDK hands the library)
         ├─ HKDF "pubky-chat/archive-wrap/v1" → wraps the archive key
         └─ HKDF "pubky-chat/inbox-wrap/v1"   → wraps each inbox key version
inbox key (X25519 HPKE, per user, random; shared through the self group; rotated when a device is removed)
archive key (symmetric, per user, random; shared through the self group; wrapped under W, and under the recovery code as a fallback)
```

- **Valid device record:** the grant verifies to the pubky, `att` names the device key, the grant is unexpired, and it isn't revoked (H3).
- **Unattested records** (Ring cookie users before R0) are marked unverified and pinned on first use.
- Message keys are never derived from identity, grant, PoP or session material.
- **Scoped keys (Pubky SDK, K6).** These are the SDK's hierarchical path keys. Their properties, as the SDK defines them:
  - **Scope follows the grant.** A directory scope (`/priv/chat/v1/`) yields a subtree seed. A file scope yields a key for that exact file. Neither reaches a sibling such as `/priv/chat/v1-evil`.
  - **Separate trees.** `/pub/` and `/priv/` derive separate trees. Chat uses `/priv/`, which adds homeserver access control to the encryption.
  - **Domain separation.** The root derivation uses a dedicated namespace, apart from the identity signing key.
  - **Stable.** The same scope always yields the same seed, so the user's pubky backup restores it.
  - **Delivery.** The signer sends the seed beside the grant, in the encrypted relay payload, never inside the grant the homeserver stores. A Passport agent (SSO H1) holds scoped seeds and derives child keys locally.
  - **File keys only.** The SDK keeps directory seeds inside its key bundle and derives keys for file paths only. W's path is a key name: nothing is stored there.
  - **Opt-in per sign-in.** The library requests keys with the SDK's V1 approval format. A signer that predates it returns a bare grant, which a V1 flow rejects. Those users sign in without keys and get the recovery-code fallback.
  - **SDK draft status ([pubky-homeserver#668](https://github.com/pubky/pubky-homeserver/pull/668), 6 Oct).** It matches the properties above. The link-secret fix is done (6 Oct: HPKE-sealed to an app-held `ek`). **Explicit `e` permission (7 Oct, [#668](https://github.com/pubky/pubky-homeserver/pull/668#issuecomment-6035403798)).** Keys are delivered only for scopes that carry the new `e` action. `r` and `w` no longer deliver keys, and `e` grants no storage access (`/pub/chat/:rwe` = storage plus keys, `/pub/chat/:e` = keys only). Signers may approve storage while declining `e`. Upgrade order: the homeserver first (older homeservers reject grants that carry `e`), then apps and signers together (older signers fail closed on `:rwe`, and approvals from the earlier draft are rejected on restore). Chat therefore requests `/priv/chat/v1/:rwe`; a user who declines `e` gets the recovery-code fallback. Deferred to a later `v2`: an agent issuing key-bearing approvals to child apps (SSO H1, Q12).
- **What the library adds.** The SDK derives stable scoped keys only. It has no purpose labels and no data-key wrapping.
  - The chat library derives every purpose key from W with HKDF and its own versioned labels, as above. A new label version is a new key.
  - The archive key and inbox keys stay random. W only wraps them. Wrapping a random key, rather than encrypting with W directly, is what allows rotation.
- **Never for MLS.** S and its derived keys never become MLS credentials, KeyPackages, HPKE init keys or epoch secrets. A deterministic key there would turn the root into a single point that reveals every conversation and would remove post-compromise security.
- **Revocation does not reach keys.** Revoking a grant stops future `/priv` reads, but a device keeps any seed and ciphertext it already downloaded. Only rotation revokes cryptographically:
  - when a device is removed, the remaining devices rotate the inbox key and send the new version through the self group, which the removed device has left;
  - the copy wrapped under W in `/priv/chat/v1/` is protected from that device only by homeserver access control;
  - spec v3 states this.

### 3.3 L1: storage layout

```
/pub/chat/v1/
  inbox.json                             {v, inbox_pk, epoch, policy}            published inbox key + spam policy
  devices/<device_id>/device.json        {v, sig_key, grant_jws, client_id, kinds_v, created_at}
  devices/<device_id>/kp/<kp_ref>        KeyPackage pool (≈20; the owner deletes each after use) + kp/last-resort
  devices/<device_id>/w/<kp_ref>         Welcome written by the sender
  devices/<device_id>/g/<tag>/<seq>      MLS PrivateMessages
  devices/<device_id>/b/<blob_id>        attachment ciphertext
  knocks/<bucket>/<knock_id>             L2 bridge knocks (sender-owned)
  groups/<tag>/c/<epoch>                 the committer's commit slot (create-only, H7)
/priv/chat/v1/                           read cursors, drafts, archive (encrypted under the archive key)
  keys/archive.json                      {v, kid, wrapped}   archive key wrapped under HKDF(S, "pubky-chat/archive-wrap/v1")
  keys/inbox/<epoch>.json                {v, epoch, wrapped} each inbox key version wrapped under HKDF(S, "pubky-chat/inbox-wrap/v1")
```

- **Archive paths are stable.** `/priv/chat/v1/` and every archive file keep their paths for life.
  - The seed is derived from the path, so moving the archive changes S and orphans every wrapped key.
  - A new layout gets a new versioned root (`/priv/chat/v2/`), with a migration that re-wraps the keys.
  - The wrap's AEAD associated data binds the owner, the path and the key version, so a wrapped key can't be moved to another path or user.
- **Recovery needs the ciphertext too.** S comes back from the pubky backup, but history only comes back if the `/priv/chat/v1/` ciphertext survives at its original paths. That means the user's homeserver, or a backup of it that keeps the paths. `chat-backup` exports both.
- **Path tags.** `tag` = `MLS-Exporter("pubky-chat/path/v1", group_id, 16)`. It rotates every epoch.
- **No pubky in any path.**
- **Padding.** Messages are padded to 256 B, 1 KiB, 4 KiB or 16 KiB. The body cap is 16 KiB.
- **Nothing under `/pub/chat/` is indexed by Nexus** (N2).

### 3.4 Discovery and scale

**L2, first-contact mailbox.**

- **What a knock is.** A fixed-size (1 KiB) **pointer**, never a body. It is sealed with HPKE to the recipient's `inbox_pk`. It contains:
  - the sender's pubky;
  - the sender's device id;
  - the `kp_ref` of the Welcome;
  - the gate proof;
  - an expiry.

  The sender's identity is visible only inside the seal (the gift-wrap / sealed-sender pattern).
- **Native target (H8, Pubky core).** The recipient's homeserver exposes `/pub/chat/v1/inbox/`. The rules:
  - write is append-only (`Action::Append`): create-only, with no overwrite or delete by writers;
  - writers must be authenticated, with per-writer rate limits and a per-prefix byte quota;
  - the policy in `inbox.json` is enforced on write;
  - the owner lists and deletes.
- **Bridge until H8 ships (B1, us).**
  - The sender writes the knock to **its own** homeserver at `knocks/<bucket>/<knock_id>`.
    - `bucket` = the first *b* bits of `H("pubky-chat/knock-bucket/v1" ‖ inbox_pk ‖ epoch)`. Epochs are weekly, and *b* is set by spec so each bucket covers at least 256 published inbox keys.
    - The record holds the ephemeral `E`, `tag = H(DH(e, inbox_pk) ‖ epoch)` and the sealed knock.
  - Only the recipient can compute `DH(inbox_sk, E)`, so only the recipient can match the tag. Observers learn the bucket, never the recipient.
  - The index (L3 service) follows event streams for `knocks/`. It answers "knocks in bucket *x* since cursor *n*" with pointers.
  - The recipient checks each tag, fetches the knock from the sender's homeserver, verifies it, then fetches the Welcome.
- **Spam gate.** Each user picks one in `inbox.json`:
  - `pow` (the default for strangers: a fixed difficulty, verified before anything renders);
  - `contacts-only`;
  - `postage` (a Paykit payment proof to the recipient's endpoint).

  Contacts and existing conversations bypass the gate. Knocks that fail the gate are discarded unseen.
- **Admission.** Knocks that pass the gate land in **Requests** (§3.7).

**L3, change-feed index (I1, us; H9 and N1, Pubky core).**

- The index follows homeserver event streams (H9) for `/pub/chat/v1/` and stores `tag → (homeserver, last_cursor, path pointer)`.
- A client sends its current tags, grouped into 16-bit prefixes, with its last cursor. The answer lists the tags that changed. The client fetches only those homeservers, from `seq`.
- Cost grows with new messages, not with contacts or follows.

**L4, resilience.**

- If the index is down, stale or untrusted, the client crawls its contacts' and conversations' homeservers directly: round-robin, with durable per-peer cursors.
- Clients also crawl a rotating sample of conversations on every sync to audit the index for omissions.

**Index rules (normative in spec v3):**

- An index stores blinded tags, cursors and pointers only. No content, no pubky-to-pubky edges, no query logs beyond rate limiting.
- Anyone can run one. Clients use several and merge their answers.
- Clients verify every answer against the homeserver: the file must exist, its MLS authentication must hold, and its tag must match. An index that omits or forges answers is down-weighted, and L4 takes over.
- Discovery never depends on Nexus or on any single index.

### 3.5 Conversations, groups, ordering

- **A DM** is an MLS group of both users' devices, `group_id = H("pubky-chat/dm/v1" ‖ sort(A, B))`. If both create it at once, the lower pubky's group wins.
- **Groups and private rooms** are MLS groups. The group context carries the name, admins and committer.
- **Committer.**
  - Commits go only to the committer user's create-only slot. Others send proposals. Application messages never wait.
  - The committer hands over by commit.
  - If the committer is gone, any admin re-creates the group with a `predecessor` reference.
  - A committer device automatically adds any newly attested device of an existing member.
- **Content** is kinds-v2 with these changes:
  - authorship comes from the MLS sender credential;
  - `chat.group.membership.v0` and `chat.group.invite.v0` become MLS proposals plus Welcome;
  - `channel_id` becomes `group_id`;
  - receipts carry the lifecycle acknowledgements.

### 3.6 Public rooms

- Rooms' layout becomes **public rooms v1**: room, membership, message, reaction and ban files, with creator bans honored by readers.
- Rooms are discovered through tags on the room URI. Messages stay room files, not posts.
- "Posting here is public" copy is mandatory.

### 3.7 Message lifecycle and product definitions

**Lifecycle (Severin):**

| State | Meaning | UI |
|---|---|---|
| accepted | Persisted to the local store the moment the user presses Send, before any network, policy or crypto work | `Queued` |
| queued | Waiting for transmission (offline, closed tab, gate proof in progress) | `Queued` |
| transmitted | Ciphertext stored on the sender's homeserver, plus a knock for first contact | `Sent` |
| acknowledged | A recipient device returned `delivered` | `Delivered` |
| (read) | A recipient read it, where both sides allow read receipts | `Read` |
| failed | Permanent refusal: blocked, gate refused, or recipient left | `Failed` with Retry |

**Settled decisions:**

1. **Offline Send.** Send is persisted first and is never lost while a tab is closed. Transmission resumes on the next app start, or through Background Sync where the browser supports it. Once transmitted, it is delivered from the homeserver mailbox whenever the recipient comes online.
2. **Anyone can message any profile,** through the L2 mailbox with spam gating. Admission policy decides Inbox versus Requests.
3. **History survives sign-out.**
   - Sign-out locks the owner-scoped encrypted store and never deletes it; signing back in as the same pubky reopens it.
   - The encrypted archive (`/priv/chat/v1/`) carries history to new devices.
4. **Multi-device is supported** through MLS devices and the self group. Every app installation is a device.
5. **Recovery covers the archive and keys.**
   - **Primary path: the user's pubky backup, through the signer.** A new device signs in with a grant covering `/priv/chat/v1/`. Its signer derives S and delivers it beside the grant. The device derives W from it and unwraps the archive key and the current inbox key from `/priv/chat/v1/keys/`, and with them restores history, contacts, read state, blocks and the inbox.
   - **Fallback: the recovery code.** It seals the same two keys for users whose signer can't yet deliver scoped keys. It is offered only while that is the case.
   - **What has to survive:** the root (the pubky backup) and the `/priv/chat/v1/` ciphertext at its original paths.
   - The new device gets a fresh, grant-attested device key. Recovery never restores MLS state.
   - Committers re-add it to every conversation automatically.
6. **Definitions:**
   - **Sent:** transmitted.
   - **Delivered:** acknowledged by at least one recipient device. Always on, content-free.
   - **Read:** the recipient's client has displayed it. On by default, switchable off. Switching it off is symmetric: you neither send nor see read receipts.
   - **Unread:** not yet displayed on any of your devices. The read cursor syncs through the self group.
   - **Requests:** conversations from senders who don't pass your admission policy. Nothing is resolved over the network until you open one. Accept moves it to the Inbox. Decline deletes it locally; the sender is never told.
   - **Mute:** a conversation or person stays fully delivered, but raises no notification or unread badge. Stored in social-specs mutes; the sender is never told.
   - **Block:** you leave the DM group, and their knocks and Welcomes are ignored. They vanish from Requests. Synced to all your devices. The sender sees `Sent`, never `Delivered`.

### 3.8 Library

- **Packages:** the Hypercolor package suite (P3), plus:
  - `pubky-chat-mls` (Rust, OpenMLS, WASM and UniFFI);
  - `@pubky/chat-transport-mls`;
  - `@pubky/chat-index-client`, which implements L2 and L3 queries, L4 crawling and answer verification.
- **Storage borrows the host's grant session** (SSO principle 7). The library never restores, refreshes or signs it out.
- **One lease per app** covers MLS state and the outbox (Web Locks on web).
- **Hosts:**
  - Hypercolor is the reference client.
  - The Shop gets listing threads and offers.
  - pubky.app hosts **Messages**.
  - Rooms runs private rooms in the browser with a browser-held grant.

### 3.9 Why not Matrix, XMTP or Nostr NIP-17

| Stack | Why we don't adopt it |
|---|---|
| **Matrix** | <ul><li>Its own identity (`@user:server`) and its own federated homeservers, which replicate room state.</li><li>A Pubky key would become a login method for a Matrix account.</li><li>E2EE is Olm/Megolm with cross-signing; MLS in Matrix is experimental.</li><li>Browser SDKs are heavy, and every user would need a second server account.</li></ul> |
| **XMTP** | <ul><li>Uses MLS through OpenMLS, which we use directly.</li><li>Brings its own inbox-ID registry and node network as the mailbox, and is built around wallet identity.</li><li>Pubky identity would sit behind XMTP's.</li><li>Messages would leave homeservers for XMTP nodes.</li></ul> |
| **Nostr NIP-17** | <ul><li>secp256k1 identities and relays as mailboxes.</li><li>NIP-44 encryption has no forward secrecy or post-compromise security, and NIP-17 has no multi-device model.</li><li>Pubky keys would need a parallel Nostr key, and relays a second infrastructure.</li></ul> |

**Orlando's "adopt now, replace later":** no. Adopting a stack now means migrating users twice and running a second identity and a second mailbox in between. The current Shop messaging stays frozen as beta until MLS ships.

**What we reuse:**

- **OpenMLS**, the cryptographic core.
- **The mailbox/relay idea** from XMTP nodes and Nostr relays, as L2 and L3, with homeservers as the store.
- **Gift wrap (NIP-59) and sealed sender (Signal)** for knocks.
- **Matrix's `since` sync token** as the L3 cursor API.

### 3.10 UX principles

1. Send always succeeds locally and shows its lifecycle state honestly (§3.7).
2. One conversation everywhere. pubky.app hosts Messages; the Shop shows its listing threads in place.
3. Trust is quiet until it changes: "new device" and "key changed" notices appear inline.
4. Strangers cost nothing until opened (P9). Public is labeled public.
5. Rooms' live layer is the baseline: presence, typing, pending → stored, paging.
6. Nothing claims reachability, delivery or recovery that the client can't observe.

## 4. Shop now and migration

**Phase 0 (Shop team, now):**

- **Already shipped** in Shop v0.6.45 ([pubky-app#182](https://github.com/BitcoinErrorLog/pubky-app/pull/182)):
  - peer key pinning, with Verify and Accept on a key change;
  - at-rest encryption of history and the outbound queue;
  - republishing the Shop's own marker when another app replaces it.
- **Freeze.** No feature work on the current messaging stack.
- **Label.** Messages are marked **beta**. New conversations are limited to buyer–seller (listing and order) and mutual-follow chats.
- **P0-1, receive cap defers** (S). The intake cap stops the checkpoint at the first message it doesn't process. The rest is processed on the next tick, and nothing is consumed unread.
- **P0-2, sign-out keeps the already-encrypted history** (S).
  - Sign-out and account switches lock the owner-scoped history and outbox and never delete them.
  - The at-rest key is owner-bound and survives sign-out.
  - Signing back in as the same pubky reopens it.
- **Dropped: Paykit scope narrowing.** Paykit state is shared per identity by design (Ben, 2 Oct), so a narrower folder scope can't isolate one app's messaging. The exposure ends when the Shop drops `/pub/paykit/:rw` after F5.
- **Parked: the handshake static-key check** on the frozen stack. Upstream, Ben's signed Noise-key proof ([pubky/paykit-rs#169](https://github.com/pubky/paykit-rs/pull/169)) closes the App Registry key swap. The handshake gap we raised on it is closed too: Ben's commits `4eda7102` and `73345917` check the peer's static key against the signed key before any transport use, on handshake completion and on restore. A mismatch fails into recovery-required, and substitution tests cover both roles and restored links. The frozen Shop stack doesn't get this check; it ends at E1/F5.
- **Cookie sessions until E1.** SSO item F2 is replaced by E1. Shop messaging stays on Ring cookie sessions until the MLS cutover, and E1 is what brings messaging to Passport and Bitkit users. Cookie removal (SSO H4) waits for E1.

**Migration (E1):**

- Each thread runs on both transports for one window: MLS when the peer publishes `/pub/chat/v1/devices/*`, otherwise the frozen Encrypted Link.
- History and outbox import locally into `chat-store`, with `event_id`s kept.
- `marketplace.chat_message.v0` maps to `chat.context.v0`.
- After the window, Encrypted-Link chat is removed (F5). The Shop deletes its `conversation-requests/*` files.

**Hypercolor** cuts over cleanly (P19).

## 5. Phased plan

| # | Item | Owner | Size | Phase |
|---|---|---|---|---|
| P0-1 | Receive cap defers instead of consuming | Shop team | S | 0 |
| P0-2 | Sign-out keeps the already-encrypted history | Shop team | S | 0 |
| P0-3 | Beta label and conversation scope | Shop team | S | 0 |
| C1 | Spec v3: MLS profile, L1 layout, knock format, index protocol and rules, lifecycle, product definitions, public rooms v1, vectors | us, with Matt and Paykit | L | 1 |
| C2 | TypeScript packages with lifecycle, lease, reactive store, durable cursors and drafts (Severin's state-consistency program) | us | L | 1 |
| C3 | `pubky-chat-mls` and `chat-transport-mls` | us | L | 1–2 |
| X1 | Independent protocol review and security audit of C1, C3, B1 and I1, plus the key wrapping, HKDF labels and recovery path of D1 | us | — | 2 |
| B1 | L2 bridge: sender-side blinded knocks, gate (`pow`, `contacts-only`, `postage`), Requests | us | M | 2 |
| I1 | L3 index service (`pubky-chat-index`), first public instance, and L4 crawl in `chat-index-client` | us | M | 2 |
| E1 | Shop on the packages, with migration (§4) | Shop team | L | 2 |
| E2 | Hypercolor on MLS | us | M | 2 |
| U1 | Usability set: timestamps, pagination, receipts, edit, 16 KiB bodies, attachments v1 | us | M | 2 |
| D1 | Self group, archive, multi-device, and recovery: random archive and inbox keys wrapped under HKDF-derived keys from W (the SDK file key under S) at stable `/priv/chat/v1/keys/` paths; recovery-code fallback; `chat-backup` export of the ciphertext with its paths | us | L | 3 |
| E3 | pubky.app Messages | pubky-app maintainers | M | 3 |
| E4 | Rooms private rooms (browser-held grant) and `pubky_ex` second implementation | Matt | L | 3 |
| B2 | Switch L2 from the bridge to the native inbox (H8) | us | S | 3 |
| U2 | Push waker, search | us | M | 4 |
| F5 | Remove Encrypted-Link chat | us | S | 4 |

### Core commitments

| # | Ask | Owner | Phase |
|---|---|---|---|
| H8 | Homeserver append-only inbox: create-only writes by any authenticated pubky, per-writer rate limits, a per-prefix quota, the policy from `inbox.json` enforced on write, owner list and delete | Pubky core | 3 |
| H9 | Event stream for indexers: a resumable homeserver change feed filtered by path prefix, with no per-connection user cap | Pubky core | 2 |
| N1 | Index hosting: a second public `pubky-chat-index` instance alongside Nexus. The software is I1 | Pubky core (hosting), us (software) | 2 |
| N2 | Nexus never indexes `/pub/chat/` | Pubky core | 1 |
| K5 | Grant `att` claim, carried through child grants | Pubky core | 1 |
| K6 | Scoped key derivation in the SDK (Rust, JS, FFI), delivered beside the grant in the encrypted relay payload, with signer support in Ring, Bitkit and Passport. The SDK side is in draft ([pubky-homeserver#668](https://github.com/pubky/pubky-homeserver/pull/668), Andrei). It gates D1's primary recovery path; until a signer supports it, its users get the recovery-code fallback | Pubky core (Andrei), Ring, Bitkit, Passport | 3 |
| H7 | Create-only conditional PUT (`If-None-Match: *`) | Pubky core | 1 |
| H3 | Grant status for verifiers | Pubky core | 1 |
| SSO | H5, H6, R0, H1 per the SSO plan. Delegated grants (H1) are still core's open item; the scoped-key work doesn't design them | Pubky core, Ring | per SSO |

### Other owned decisions

| Decision | Owner | Phase |
|---|---|---|
| Paykit keeps Encrypted Links for payments. Chat references `paykit.payment_*` kinds and never carries payment settlement itself | Paykit | 1 |
| No Paykit work for chat or browser payments: the storage interface (Y1), the WASM package (Y2) and a custom-message API are withdrawn. Payments need nothing in the browser beyond a public read, and chat runs on MLS (Ben, 2 Oct) | Paykit | — |
| Public rooms v1 standardizes Rooms' layout, and pubky.app renders rooms from it | Matt, us | 1 |
| Member-list discovery for rooms uses the L3 index | Matt | 2 |
| Membership changes wait for the committer; handover and re-creation bound the stall | us | 1 |
| Abuse in public rooms: the creator moderates through bans; reports go to the creator as a private message; there is no central moderation | Matt | 3 |
| pubky-chat moves to the `pubky` org once Pubky core signs off spec v3 | us, Pubky core | 2 |

## 6. Hub

**[pubky/pubky-chat](https://github.com/pubky/pubky-chat)** is the hub:

- it holds the spec, vectors and CI;
- it is public, MIT and BitcoinErrorLog-owned;
- its layout has room for `packages/`, `native/`, the Rust crate and the index service.

Alternatives were rejected:

- **pubky-marketplace:** it is organized around the Shop.
- **paykit-rs:** it is the payments stack, and outside BitcoinErrorLog.
- **pubky-rooms:** a single app.
- **hypercolor-web:** an app; its ADRs are inputs, not the shared contract.

This plan lives in `docs/`, spec v3 in `spec/`, and the index service in `index/`.
