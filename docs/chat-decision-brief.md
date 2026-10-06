# Chat: decisions and next steps

The full plan, with owners and phases, is the [chat unification plan](https://github.com/pubky/pubky-chat/blob/main/docs/chat-unification-plan.md).

## What we decided

- **Encryption:** MLS (through OpenMLS) for every private conversation, one-to-one included.
- **Devices:** the user's grant vouches for each device's key, and every app install counts as its own device. That gives multiple devices and group chats.
- **One shared library:** the Shop, pubky.app, Rooms and Hypercolor all use the same chat library.

## How discovery and scale work: four layers

| Layer | Job | How it works |
|---|---|---|
| **L1: Storage** | Source of truth | Homeservers hold everything. Each author keeps their encrypted messages under `/pub/chat/v1/`, at rotating, padded paths. |
| **L2: Mailbox** | First contact | Anyone can send any profile a sealed "knock", protected against spam by proof-of-work, contacts-only mode or a small Paykit payment. Until Pubky core builds a native inbox, the sender writes the knock on their own homeserver under a tag only the recipient can compute. |
| **L3: Change feed** | Scale | An index tracks which conversations changed. Clients fetch only those, so cost grows with new messages, not with the number of contacts. This ends the 30+ minute sync. |
| **L4: Fallback** | Resilience | If the index is down or untrusted, clients check their contacts directly. |

Rules for the index:
- It only speeds things up. Nothing depends on it.
- It never stores message content or who follows whom.
- Anyone can run one, and clients can use several.
- Clients check every answer against the homeservers.

## Answers to Severin's audit

- Each of the 12 root causes maps to the part of the design or the phase that removes it. The table is in the plan.
- We adopted his four programs, his message lifecycle (accepted, queued, transmitted, acknowledged) and his product definitions.
- His six open decisions are settled:

| Question | Decision |
|---|---|
| What does Send promise offline? | The message is saved locally first and delivered through the mailbox. It is never lost when tabs close. |
| Can anyone message any profile? | Yes, through the L2 mailbox, with spam protection. |
| Does history survive sign-out? | Yes, through the encrypted archive. |
| Multiple devices? | Yes, through the MLS device model. |
| What does recovery cover? | The archive plus key recovery. |
| What do Sent, Read, Requests, Mute and Block mean? | Each has one definition, used everywhere. |

## Answer to Orlando's question

We won't adopt Matrix, XMTP or Nostr. Each brings its own identity and mailbox network, which would push Pubky identity into second place.

From them we reuse OpenMLS and the patterns they've proven:
- mailboxes;
- sealed sender;
- sync cursors.

## Shop, now (Phase 0, Shop team)

- Current Shop messaging is frozen and labeled beta, for buyer–seller and mutual-follow chats.
- Only two fixes ship:
  1. A backlog of incoming messages waits instead of being dropped.
  2. Sign-out keeps the history, which is already encrypted.
- Already live in Shop v0.6.45:
  - checking contacts' messaging keys, with Verify and Accept;
  - encrypted history on the device.

## What we need from Pubky core

| Phase | Ask |
|---|---|
| 1 | The `att` grant claim, create-only writes, grant status, and Nexus skipping `/pub/chat/` |
| 2 | A stream of homeserver updates that indexers can follow, and hosting the chat index alongside Nexus |
| 3 | The native append-only inbox, with a spam policy |

## Next steps

| Owner | Work |
|---|---|
| Us | Spec v3 and the TypeScript packages (Phase 1), then the MLS transport, the mailbox bridge and the index (Phase 2) |
| Shop team | The two Phase 0 fixes now, then the switch to the new library in Phase 2 |
| Matt | Public rooms v1, then private rooms |
| Paykit | Y1 and Y2 for payments |
