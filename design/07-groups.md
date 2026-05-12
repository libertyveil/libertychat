# Group chats with MLS

## Why MLS

Most messengers built groups using ad-hoc protocols extended from one-to-one designs:

- **Signal Sender Keys**: each member sends encrypted to all others, with a shared sender-key for efficiency. Adding/removing members requires re-keying, which doesn't scale beyond ~150-1000 members.
- **Matrix Megolm**: similar model, member changes are expensive.
- **SimpleX**: each pair of group members has a 1:1 connection; group message is sent N-1 times. Doesn't scale beyond ~50 members.

MLS (RFC 9420) was designed by IETF to solve exactly this problem. Properties:

- **Logarithmic key updates** for member changes (TreeKEM)
- Groups of thousands of members feasible
- **Forward Secrecy** preserved
- **Post-Compromise Security** after every epoch
- **Authentication** of group operations
- Standardized, formally analyzed

LibertyChat uses MLS for all group chats from day one.

## Implementation

Use AWS's `mls-rs` Rust library. It's:

- Production-grade (used in AWS Wickr)
- Audited
- Implements RFC 9420 strictly
- Designed for embedding in client apps

## Group lifecycle

### Creating a group

1. Creator generates a new MLS group with their device key
2. Creator invites initial members; each invite is an MLS Welcome message
3. Welcome messages delivered via members' existing 1:1 mailbox channels
4. Each invited member processes the Welcome, joins the group state

### Adding a member

1. An admin proposes the addition (MLS `Add` proposal)
2. Other admins or the group owner commit the proposal
3. New epoch keys derived; old keys retired
4. New member receives Welcome message with current group state

### Removing a member

1. An admin proposes removal (MLS `Remove`)
2. Commit transitions to new epoch
3. Removed member can no longer decrypt new messages
4. Past messages remain readable to remaining members (no retroactive secrecy from co-members)

### Sending a message

1. Encrypt message with current epoch's group key
2. Send to all members via the group's mailbox queue (one upload, N retrievals)
3. MLS state advances forward (symmetric ratchet within an epoch)

## Group identity

A group is identified by a unique group ID (hash of the initial state). Groups have:

- **Display name** (mutable)
- **Avatar / icon**
- **Description**
- **Invite link** (long-lived, optional, for new members to join)

Group state is stored client-side and synced via MLS commits.

## Roles

Roles are application-level (built on top of MLS):

- **Owner**: can add/remove admins
- **Admin**: can add/remove members, edit group metadata
- **Member**: can send messages, leave the group

Permissions enforced by clients (signed by group state). Misbehaving clients can be detected and removed.

## Group calls

Group calls in a group chat use the SFU model (see [`05-realtime.md`](05-realtime.md)) with **SFrame** for E2E encryption. The group's MLS keys derive call session keys.

## Privacy

The mailbox server hosting the group queue sees:

- Member count (queue subscribers)
- Message timing and frequency
- Encrypted message blobs

It does not see:

- Member identities (subscribers authenticate per-queue, not per-user-identity)
- Message contents
- Group metadata (name, description, etc.)

## Compatibility

MLS is an IETF standard. Other MLS-compliant messengers can theoretically interoperate at the cryptographic layer. The application protocol on top (group metadata, role definitions, file attachments) is LibertyChat-specific but documented openly.

Future work: define a federated group bridge so groups can span LibertyChat and other MLS-using systems (Matrix, Wire).

## Comparison

| | LibertyChat | SimpleX | Matrix (Megolm) | Matrix (MLS, in progress) |
|---|---|---|---|---|
| Group crypto | MLS RFC 9420 | per-pair 1:1 (Sesame variant) | Megolm | MLS |
| Max practical group size | thousands | ~50 | ~1000 | thousands |
| Add/remove cost | logarithmic | linear | linear | logarithmic |
| Forward Secrecy | yes | yes | per-message | yes |
| Standardization | IETF | proprietary | proprietary | IETF |
| Federation possible | yes (planned) | no | yes | yes |
