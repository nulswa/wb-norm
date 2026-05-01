<div align="center">

```
██╗    ██╗██████╗       ███╗   ██╗ ██████╗ ██████╗ ███╗   ███╗
██║    ██║██╔══██╗      ████╗  ██║██╔═══██╗██╔══██╗████╗ ████║
██║ █╗ ██║██████╔╝█████╗██╔██╗ ██║██║   ██║██████╔╝██╔████╔██║
██║███╗██║██╔══██╗╚════╝██║╚██╗██║██║   ██║██╔══██╗██║╚██╔╝██║
╚███╔███╔╝██████╔╝      ██║ ╚████║╚██████╔╝██║  ██║██║ ╚═╝ ██║
 ╚══╝╚══╝ ╚═════╝       ╚═╝  ╚═══╝ ╚═════╝ ╚═╝  ╚═╝╚═╝     ╚═╝
```

**A hardened, normalized fork of Baileys — WhatsApp Web API for Node.js**

[![License: MIT](https://img.shields.io/badge/license-GPL--3.0-0a0a0a?style=flat-square&labelColor=111)](LICENSE)
[![Node.js](https://img.shields.io/badge/node-%3E%3D18.0.0-0a0a0a?style=flat-square&labelColor=111)](https://nodejs.org)
[![WhatsApp](https://img.shields.io/badge/WhatsApp-6.x-0a0a0a?style=flat-square&labelColor=111)](https://www.whatsapp.com)

</div>

---

> **wb-norm** is an independent, normalized fork of the [Baileys](https://github.com/WhiskeySockets/Baileys) library. It is not affiliated with, endorsed by, or officially connected to WhatsApp Inc. or Meta Platforms. Use responsibly and in accordance with WhatsApp's Terms of Service.

---

## Table of Contents

- [Why wb-norm](#why-wb-norm)
- [Core Features](#core-features)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Connecting to WhatsApp](#connecting-to-whatsapp)
  - [QR Code Authentication](#qr-code-authentication)
  - [Pairing Code Authentication](#pairing-code-authentication)
  - [Receiving Full Message History](#receiving-full-message-history)
- [Socket Configuration](#socket-configuration)
  - [Group Metadata Caching](#group-metadata-caching)
  - [Message Store & Poll Decryption](#message-store--poll-decryption)
  - [Notification Mode](#notification-mode)
- [Persisting Auth State](#persisting-auth-state)
- [In-Memory Data Store](#in-memory-data-store)
- [WhatsApp ID Reference](#whatsapp-id-reference)
- [Sending Messages](#sending-messages)
  - [Text & Rich Messages](#text--rich-messages)
  - [Media Messages](#media-messages)
  - [Interactive Messages](#interactive-messages)
- [Modifying Messages](#modifying-messages)
- [Media Utilities](#media-utilities)
- [Managing Chats](#managing-chats)
- [Groups](#groups)
- [Privacy & Security](#privacy--security)
- [Broadcasts & Stories](#broadcasts--stories)
- [Presence & Status](#presence--status)
- [Low-Level WebSocket Events](#low-level-websocket-events)
- [Best Practices](#best-practices)
- [License](#license)

---

## Why wb-norm

Baileys is a powerful library — but its raw form ships with known edge cases around JID normalization in group contexts. **wb-norm** exists to address that, without diverging from the upstream architecture.

| Issue | Upstream Baileys | wb-norm |
|---|---|---|
| `@lid` → `@pn` resolution in groups | ❌ Inconsistent | ✅ Normalized |
| Multi-device support | ✅ | ✅ |
| WhatsApp 6.x protocol | ✅ | ✅ |
| Drop-in replacement | — | ✅ Same API surface |

If your bot was losing participant mentions, misidentifying senders in groups, or failing on `@lid` JIDs, wb-norm was built to fix that.

---

## Core Features

- **`@lid` → `@pn` normalization** — Resolves the group JID mismatch that breaks mention parsing and participant lookups
- **Full Multi-Device support** — Works with WhatsApp's current multi-device architecture
- **All message types** — Text, image, video, audio, stickers, polls, reactions, pins, forwards, and more
- **End-to-end encrypted** — Signal Protocol throughout, identical to the official client
- **Link preview generation** — Via `link-preview-js` integration
- **Media streaming** — Never loads full buffers into memory; encrypts and streams
- **TypeScript-first** — Fully typed, with accurate type exports

---

## Installation

```bash
# Stable release
npm install wb-norm

# Latest edge build
npm install wb-norm@latest
```

```javascript
// CommonJS
const { default: makeWASocket } = require("wb-norm")

// ES Modules
import makeWASocket from "wb-norm"
```

---

## Quick Start

A minimal bot that connects to WhatsApp and echoes every message:

```javascript
const {
  default: makeWASocket,
  DisconnectReason,
  useMultiFileAuthState
} = require("wb-norm")
const { Boom } = require("@hapi/boom")

async function start() {
  const { state, saveCreds } = await useMultiFileAuthState("./auth")

  const sock = makeWASocket({
    auth: state,
    printQRInTerminal: true,
    browser: ["wb-norm", "Desktop", "6.0"]
  })

  // ── Connection lifecycle ──────────────────────────────────────────────────
  sock.ev.on("connection.update", ({ connection, lastDisconnect }) => {
    if (connection === "close") {
      const code = (lastDisconnect?.error)?.output?.statusCode
      const retry = code !== DisconnectReason.loggedOut
      console.log("Connection closed —", lastDisconnect?.error?.message)
      if (retry) start()
    } else if (connection === "open") {
      console.log("✓ Connected to WhatsApp")
    }
  })

  // ── Incoming messages ─────────────────────────────────────────────────────
  sock.ev.on("messages.upsert", async ({ messages }) => {
    for (const m of messages) {
      if (!m.message) continue
      console.log("↓", m.key.remoteJid, JSON.stringify(m.message))

      await sock.sendMessage(m.key.remoteJid, {
        text: "Received ✓"
      })
    }
  })

  sock.ev.on("creds.update", saveCreds)
}

start()
```

---

## Connecting to WhatsApp

WhatsApp's multi-device API allows wb-norm to authenticate as a secondary linked device — either via QR scan or pairing code.

### QR Code Authentication

```javascript
const { default: makeWASocket, Browsers } = require("wb-norm")

const sock = makeWASocket({
  browser: Browsers.ubuntu("MyBot"),
  printQRInTerminal: true
})
```

A QR code will render in your terminal. Scan it from **WhatsApp → Linked Devices → Link a Device**.

> **Tip:** `Browsers` exposes named presets — `ubuntu`, `macOS`, `windows`, and more. These affect what metadata WhatsApp sees for your session.

---

### Pairing Code Authentication

Pairing codes let you link without scanning a QR. They are limited to one device per code.

The phone number must be provided in full E.164 format, **without** `+`, spaces, dashes, or parentheses.

```javascript
const { default: makeWASocket } = require("wb-norm")

const sock = makeWASocket({ printQRInTerminal: false })

if (!sock.authState.creds.registered) {
  const phoneNumber = "15551234567" // E.164, no + prefix

  // Standard pairing — WhatsApp generates the code
  const code = await sock.requestPairingCode(phoneNumber)
  console.log("Pairing code:", code)

  // Custom pairing — you choose the 8-character code
  const customCode = await sock.requestPairingCode(phoneNumber, "WBNORM01")
  console.log("Custom pairing code:", customCode)
}
```

---

### Receiving Full Message History

To sync the complete message history on first connection:

```javascript
const { default: makeWASocket, Browsers } = require("wb-norm")

const sock = makeWASocket({
  browser: Browsers.macOS("Desktop"), // Desktop browser = more history
  syncFullHistory: true
})
```

History arrives in the `messaging.history-set` event after the initial connection handshake completes.

---

## Socket Configuration

### Group Metadata Caching

Group metadata (participants, subject, description) is fetched frequently. Without caching, every lookup hits the WhatsApp API and introduces latency. The recommended pattern:

```javascript
const { default: makeWASocket } = require("wb-norm")
const NodeCache = require("node-cache")

// TTL: 5 minutes, no cloning (saves memory)
const groupCache = new NodeCache({ stdTTL: 300, useClones: false })

const sock = makeWASocket({
  cachedGroupMetadata: async (jid) => groupCache.get(jid)
})

// Keep the cache warm when group state changes
sock.ev.on("groups.update", async ([event]) => {
  groupCache.set(event.id, await sock.groupMetadata(event.id))
})

sock.ev.on("group-participants.update", async (event) => {
  groupCache.set(event.id, await sock.groupMetadata(event.id))
})
```

---

### Message Store & Poll Decryption

Two features depend on a `getMessage` implementation: retry delivery and poll vote decryption. You supply this function; wb-norm calls it when needed.

```javascript
const sock = makeWASocket({
  getMessage: async (key) => {
    // Return the full WAMessage for this key from your store, or undefined
    return await myStore.findMessage(key.id)
  }
})
```

Without this, poll updates will not decrypt correctly.

---

### Notification Mode

By default, an active desktop session suppresses push notifications on the phone. Set the client offline to restore them:

```javascript
const sock = makeWASocket({
  markOnlineOnConnect: false
})
```

You can also toggle this at runtime:

```javascript
await sock.sendPresenceUpdate("unavailable")
```

---

## Persisting Auth State

`useMultiFileAuthState` writes credentials to a local directory. This is fine for development and simple deployments.

```javascript
const { default: makeWASocket, useMultiFileAuthState } = require("wb-norm")

async function connect() {
  const { state, saveCreds } = await useMultiFileAuthState("./auth")

  const sock = makeWASocket({ auth: state })
  sock.ev.on("creds.update", saveCreds)
}
```

> **Production note:** For anything beyond a single-process bot, implement your own auth state backed by a database (PostgreSQL, MongoDB, Redis). Signal keys must be read and written atomically — partial updates corrupt sessions.

---

## In-Memory Data Store

wb-norm does not ship a default persistent store. A simple in-memory store is available for development:

```javascript
const { default: makeWASocket, makeInMemoryStore } = require("wb-norm")

const store = makeInMemoryStore({})

// Optionally persist to disk between restarts
store.readFromFile("./store.json")
setInterval(() => store.writeToFile("./store.json"), 15_000)

const sock = makeWASocket({})
store.bind(sock.ev)

sock.ev.on("chats.upsert", () => {
  console.log("Chats:", store.chats.all())
})

sock.ev.on("contacts.upsert", () => {
  console.log("Contacts:", Object.values(store.contacts))
})
```

> **Warning:** The in-memory store keeps your full chat history in RAM. For bots handling large group volumes, implement a purpose-built store.

---

## WhatsApp ID Reference

Every entity in WhatsApp — person, group, broadcast list — has a unique JID (Jabber ID).

| Entity | Format | Example |
|---|---|---|
| User | `[country][number]@s.whatsapp.net` | `15551234567@s.whatsapp.net` |
| Group | `[creation_ts]-[random]@g.us` | `1234567890-987654321@g.us` |
| Broadcast list | `[creation_ts]@broadcast` | `1704067200@broadcast` |
| Status/Stories | `status@broadcast` | `status@broadcast` |

---

## Sending Messages

All messages go through a single function:

```javascript
await sock.sendMessage(jid, content, options)
```

- `jid` — destination (user JID, group JID, or broadcast)
- `content` — the message payload (type-specific, see below)
- `options` — optional: `{ quoted, ephemeralExpiration, ... }`

---

### Text & Rich Messages

#### Plain text

```javascript
await sock.sendMessage(jid, { text: "Hello, world." })
```

#### Reply / quote

Works with any message type:

```javascript
await sock.sendMessage(jid, { text: "Got it." }, { quoted: incomingMessage })
```

#### Mention users

`@number` in the text is visual only — the `mentions` array drives actual mention delivery:

```javascript
await sock.sendMessage(jid, {
  text: "Hey @15551234567, you're up.",
  mentions: ["15551234567@s.whatsapp.net"]
})
```

#### Forward a message

```javascript
const original = await myStore.findMessage(someKey)
await sock.sendMessage(jid, { forward: original })
```

#### Location

```javascript
await sock.sendMessage(jid, {
  location: {
    degreesLatitude: 37.7749,
    degreesLongitude: -122.4194
  }
})
```

#### Contact card

```javascript
const vcard = [
  "BEGIN:VCARD",
  "VERSION:3.0",
  "FN:Ada Lovelace",
  "ORG:Analytical Engine Co.;",
  "TEL;type=CELL;type=VOICE;waid=15551234567:+1 555 123 4567",
  "END:VCARD"
].join("\n")

await sock.sendMessage(jid, {
  contacts: {
    displayName: "Ada Lovelace",
    contacts: [{ vcard }]
  }
})
```

#### Reaction

```javascript
await sock.sendMessage(jid, {
  react: {
    text: "🔥",  // empty string removes the reaction
    key: targetMessage.key
  }
})
```

#### Pin a message

| Duration | Seconds |
|---|---|
| 24 hours | `86400` |
| 7 days | `604800` |
| 30 days | `2592000` |

```javascript
await sock.sendMessage(jid, {
  pin: {
    type: 1,        // 0 = unpin
    time: 604800,   // 7 days
    key: targetMessage.key
  }
})
```

#### Poll

```javascript
await sock.sendMessage(jid, {
  poll: {
    name: "Preferred deployment strategy?",
    values: ["Blue-green", "Canary", "Rolling", "Recreate"],
    selectableCount: 1,
    toAnnouncementGroup: false
  }
})
```

#### Link preview

Install `link-preview-js` first (`npm install link-preview-js`), then send text containing the URL — preview is generated automatically:

```javascript
await sock.sendMessage(jid, {
  text: "Check out wb-norm: https://npmjs.com/package/wb-norm"
})
```

---

### Media Messages

For all media types, the source can be a `Buffer`, a `{ url: "..." }` object, or a `{ stream: ReadableStream }`. **Prefer URL or stream** — wb-norm streams and encrypts without loading the full asset into memory.

#### Image

```javascript
await sock.sendMessage(jid, {
  image: { url: "./photo.jpg" },
  caption: "Launch day."
})
```

#### Video

```javascript
await sock.sendMessage(jid, {
  video: { url: "./demo.mp4" },
  caption: "Watch this.",
  ptv: false   // true = video note (circle)
})
```

#### GIF (as looping video)

WhatsApp does not support `.gif` natively. Send as `.mp4` with `gifPlayback: true`:

```javascript
await sock.sendMessage(jid, {
  video: fs.readFileSync("./animation.mp4"),
  gifPlayback: true
})
```

#### Audio

For voice notes, encode with ffmpeg first:
```bash
ffmpeg -i input.mp4 -avoid_negative_ts make_zero -ac 1 -codec:a libopus output.ogg
```

```javascript
await sock.sendMessage(jid, {
  audio: { url: "./voice.ogg" },
  mimetype: "audio/ogg; codecs=opus",
  ptt: true  // true = voice note UI, false = audio player
})
```

#### View once

Add `viewOnce: true` to any image, video, or audio message:

```javascript
await sock.sendMessage(jid, {
  image: { url: "./secret.jpg" },
  caption: "This disappears after viewing.",
  viewOnce: true
})
```

---

### Interactive Messages

#### Decrypt poll votes

Poll votes arrive encrypted in `messages.update`. Decrypt them with a `getMessage` implementation:

```javascript
sock.ev.on("messages.update", async (events) => {
  for (const { key, update } of events) {
    if (!update.pollUpdates) continue

    const original = await myStore.findMessage(key)
    if (!original) continue

    const tally = getAggregateVotesInPollMessage({
      message: original,
      pollUpdates: update.pollUpdates
    })

    console.log("Poll results:", tally)
  }
})
```

---

## Modifying Messages

#### Delete for everyone

```javascript
const sent = await sock.sendMessage(jid, { text: "Sent by mistake." })
await sock.sendMessage(jid, { delete: sent.key })
```

> To delete only for yourself, use `chatModify` — see [Managing Chats](#managing-chats).

#### Edit a sent message

```javascript
await sock.sendMessage(jid, {
  text: "Corrected content.",
  edit: sentMessage.key
})
```

---

## Media Utilities

#### Download received media

```javascript
const { downloadMediaMessage, getContentType } = require("wb-norm")
const { createWriteStream } = require("fs")

sock.ev.on("messages.upsert", async ({ messages: [m] }) => {
  if (!m.message) return

  const type = getContentType(m.message)
  if (type !== "imageMessage") return

  const stream = await downloadMediaMessage(m, "stream", {}, {
    logger,
    reuploadRequest: sock.updateMediaMessage
  })

  stream.pipe(createWriteStream("./download.jpg"))
})
```

#### Re-upload expired media

WhatsApp periodically expires media from its CDN. To re-upload before forwarding:

```javascript
await sock.updateMediaMessage(message)
```

#### Auto-thumbnail generation

Thumbnails are generated automatically for images and stickers if you have **jimp** or **sharp** installed. Video thumbnails require **ffmpeg** in `PATH`.

```bash
npm install sharp   # or: npm install jimp
```

---

## Managing Chats

WhatsApp uses an encrypted app-state sync protocol for chat modifications. Incorrect updates can log out all linked devices — use these with care.

#### Archive / unarchive

```javascript
const last = await myStore.getLastMessage(jid)
await sock.chatModify({ archive: true, lastMessages: [last] }, jid)
await sock.chatModify({ archive: false, lastMessages: [last] }, jid)
```

#### Mute / unmute

| Duration | Milliseconds |
|---|---|
| 8 hours | `28800000` |
| 7 days | `604800000` |
| Remove mute | `null` |

```javascript
await sock.chatModify({ mute: 28800000 }, jid)   // mute 8h
await sock.chatModify({ mute: null }, jid)          // unmute
```

#### Mark read / unread

```javascript
const last = await myStore.getLastMessage(jid)
await sock.chatModify({ markRead: false, lastMessages: [last] }, jid)
```

#### Delete message (self only)

```javascript
await sock.chatModify({
  clear: {
    messages: [{ id: "MSG_ID", fromMe: true, timestamp: "1700000000" }]
  }
}, jid)
```

#### Delete entire chat

```javascript
const last = await myStore.getLastMessage(jid)
await sock.chatModify({
  delete: true,
  lastMessages: [{ key: last.key, messageTimestamp: last.messageTimestamp }]
}, jid)
```

#### Pin / unpin

```javascript
await sock.chatModify({ pin: true }, jid)
await sock.chatModify({ pin: false }, jid)
```

#### Star / unstar messages

```javascript
await sock.chatModify({
  star: {
    messages: [{ id: "MSG_ID", fromMe: true }],
    star: true   // false = unstar
  }
}, jid)
```

#### Disappearing messages

| Duration | Seconds |
|---|---|
| Off | `0` |
| 24 hours | `86400` |
| 7 days | `604800` |
| 90 days | `7776000` |

```javascript
const { WA_DEFAULT_EPHEMERAL } = require("wb-norm") // 604800

// Enable for a chat
await sock.sendMessage(jid, { disappearingMessagesInChat: WA_DEFAULT_EPHEMERAL })

// Send one message as ephemeral
await sock.sendMessage(jid, { text: "Gone in 7 days." }, {
  ephemeralExpiration: WA_DEFAULT_EPHEMERAL
})

// Disable
await sock.sendMessage(jid, { disappearingMessagesInChat: false })
```

---

## Groups

All write operations (adding members, changing settings) require admin privileges.

#### Create a group

```javascript
const result = await sock.groupCreate("Engineering", [
  "15551111111@s.whatsapp.net",
  "15552222222@s.whatsapp.net"
])
console.log("Created:", result.gid)
```

#### Manage participants

```javascript
// Actions: "add" | "remove" | "promote" | "demote"
await sock.groupParticipantsUpdate(
  jid,
  ["15551111111@s.whatsapp.net"],
  "add"
)
```

#### Update group info

```javascript
await sock.groupUpdateSubject(jid, "Engineering — Q3")
await sock.groupUpdateDescription(jid, "Internal planning channel.")
```

#### Group settings

```javascript
// Restrict messaging to admins only
await sock.groupSettingUpdate(jid, "announcement")
// Open messaging to all members
await sock.groupSettingUpdate(jid, "not_announcement")
// Lock settings (admins only)
await sock.groupSettingUpdate(jid, "locked")
// Unlock settings (all members)
await sock.groupSettingUpdate(jid, "unlocked")
```

#### Invite links

```javascript
// Get current invite link code (not the full URL)
const code = await sock.groupInviteCode(jid)
console.log(`https://chat.whatsapp.com/${code}`)

// Revoke and generate a new code
const newCode = await sock.groupRevokeInvite(jid)

// Join via code
await sock.groupAcceptInvite(code)

// Get group info from a code (without joining)
const info = await sock.groupGetInviteInfo(code)
```

#### Group metadata

```javascript
const meta = await sock.groupMetadata(jid)
console.log(meta.subject, meta.desc, meta.participants)
```

#### Join requests

```javascript
// List pending requests
const pending = await sock.groupRequestParticipantsList(jid)

// Approve or reject
await sock.groupRequestParticipantsUpdate(jid, ["15551111111@s.whatsapp.net"], "approve")
await sock.groupRequestParticipantsUpdate(jid, ["15552222222@s.whatsapp.net"], "reject")
```

#### Member add mode

```javascript
await sock.groupMemberAddMode(jid, "admin_add")     // only admins can add
await sock.groupMemberAddMode(jid, "all_member_add") // anyone can add
```

#### Ephemeral mode for group

```javascript
await sock.groupToggleEphemeral(jid, 86400)  // 24h, or 0 to disable
```

#### Leave group

```javascript
await sock.groupLeave(jid)
```

---

## Privacy & Security

#### Block / unblock

```javascript
await sock.updateBlockStatus(jid, "block")
await sock.updateBlockStatus(jid, "unblock")
```

#### Fetch blocklist

```javascript
const blocked = await sock.fetchBlocklist()
```

#### Privacy settings

```javascript
const settings = await sock.fetchPrivacySettings(true)
```

Update individual settings — valid values are `"all"`, `"contacts"`, `"contact_blacklist"`, or `"none"` (where applicable):

```javascript
await sock.updateLastSeenPrivacy("contacts")
await sock.updateOnlinePrivacy("match_last_seen")
await sock.updateProfilePicturePrivacy("contacts")
await sock.updateStatusPrivacy("contacts")
await sock.updateReadReceiptsPrivacy("none")         // disables read receipts
await sock.updateGroupsAddPrivacy("contacts")
```

#### Default disappearing mode

```javascript
await sock.updateDefaultDisappearingMode(604800) // 7 days for all new chats
```

---

## Broadcasts & Stories

```javascript
await sock.sendMessage(
  "status@broadcast",
  {
    image: { url: "./banner.jpg" },
    caption: "Shipped. 🚀"
  },
  {
    backgroundColor: "#0a0a0a",
    font: 1,
    statusJidList: ["15551111111@s.whatsapp.net"],
    broadcast: true
  }
)
```

`statusJidList` — explicit list of recipients for the story.
`broadcast: true` — required to activate broadcast mode.

#### Broadcast list info

```javascript
const list = await sock.getBroadcastListInfo("1234567890@broadcast")
console.log(list.name, list.recipients)
```

---

## Presence & Status

#### Read messages

```javascript
await sock.readMessages([message.key])
```

#### Send presence

Valid values: `"available"`, `"composing"`, `"recording"`, `"paused"`, `"unavailable"`.

Presence updates expire after ~10 seconds.

```javascript
await sock.sendPresenceUpdate("composing", jid)  // typing indicator
await sock.sendPresenceUpdate("available", jid)  // online
await sock.sendPresenceUpdate("unavailable")      // go offline
```

#### Subscribe to someone's presence

```javascript
sock.ev.on("presence.update", (update) => {
  console.log(update) // { id, presences: { [jid]: { lastKnownPresence } } }
})
await sock.presenceSubscribe(jid)
```

#### Profile & status queries

```javascript
// Check if a number is on WhatsApp
const [result] = await sock.onWhatsApp("15551234567@s.whatsapp.net")
if (result.exists) console.log("Registered as:", result.jid)

// Get status text
const status = await sock.fetchStatus(jid)

// Profile picture (low or high resolution)
const url = await sock.profilePictureUrl(jid, "image") // omit second arg for low-res

// Business profile
const biz = await sock.getBusinessProfile(jid)
console.log(biz.description, biz.category)
```

#### Update your profile

```javascript
await sock.updateProfileStatus("Building with wb-norm.")
await sock.updateProfileName("My Bot")
await sock.updateProfilePicture(jid, { url: "./avatar.jpg" })
await sock.removeProfilePicture(jid)
```

#### Fetch message history

```javascript
const oldest = await myStore.getOldestMessage(jid)
await sock.fetchMessageHistory(50, oldest.key, oldest.messageTimestamp)
// Results arrive in: messaging.history-set
```

#### Reject incoming calls

```javascript
sock.ev.on("call", async ([call]) => {
  await sock.rejectCall(call.id, call.from)
})
```

---

## Low-Level WebSocket Events

wb-norm exposes the raw WhatsApp binary protocol via `sock.ws`. This lets you register callbacks for any frame tag:

```javascript
// Any frame with tag "edge_routing"
sock.ws.on("CB:edge_routing", (node) => {
  console.log(node)
})

// Frame with tag and specific id attribute
sock.ws.on("CB:edge_routing,id:abcd", (node) => {})

// Frame with tag, id, and child tag
sock.ws.on("CB:edge_routing,id:abcd,routing_info", (node) => {})
```

Each frame (`BinaryNode`) has three fields:

- `tag` — the frame type identifier
- `attrs` — key/value metadata
- `content` — payload (Buffer, string, or child nodes)

To inspect raw frames, enable debug logging:

```javascript
const P = require("pino")
const sock = makeWASocket({
  logger: P({ level: "debug" })
})
```

---

## Best Practices

**Reconnection** — Always implement auto-reconnect using the `DisconnectReason` enum. Log out on `loggedOut`, reconnect on all other codes.

**Auth management** — Store credentials in an encrypted database for production. Never commit auth files to version control.

**Rate limiting** — WhatsApp enforces rate limits. Add jitter and back-off to message sends. Automated spam results in permanent bans.

**Group caching** — Always use `cachedGroupMetadata`. Fetching group metadata on every incoming group message is the single most common cause of API bans in high-traffic bots.

**Error handling** — Wrap all socket calls in try/catch. `sock.sendMessage` can throw on network errors, JID validation failures, or media upload failures.

**Memory** — For production, never use `makeInMemoryStore` directly. Build a store backed by SQLite, PostgreSQL, or Redis and implement a `getMessage` that reads from it.

**Presence** — Set `markOnlineOnConnect: false` if your bot should receive push notifications. Active desktop presence suppresses them.

```javascript
// Minimal production-grade error handler
async function safeSend(sock, jid, content, options = {}) {
  try {
    return await sock.sendMessage(jid, content, options)
  } catch (err) {
    console.error(`Send failed → ${jid}:`, err.message)
    return null
  }
}
```

---

## Utility Functions

| Function | Description |
|---|---|
| `getContentType(message)` | Returns the type key of a WAMessage |
| `getDevice(message)` | Returns the originating device type |
| `makeCacheableSignalKeyStore(store, logger)` | Wraps a key store with LRU caching for better auth performance |
| `downloadMediaMessage(msg, type, opts, ctx)` | Downloads and decrypts media as Buffer or ReadableStream |
| `getAggregateVotesInPollMessage(opts)` | Aggregates decrypted poll vote updates |

---

## License

Distributed under the **GNU General Public License v3.0**. See [LICENSE](LICENSE) for the full terms.

wb-norm is a fork and does not claim any affiliation with the original Baileys authors or WhatsApp Inc. All modifications are subject to the GPL-3.0 license.

---

<div align="center">

**wb-norm** — Normalized. Hardened. Ready.

</div>
