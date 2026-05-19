# 23 - QR Client Integration

## Overview

This guide explains how to embed WhatsApp QR code authentication into your own frontend applications so your clients can connect their WhatsApp numbers.

## Flow

```
1. Your app  →  POST /api/sessions        →  creates a session
2. Your app  →  connects WebSocket        →  listens for events
3. OpenWA    →  emits "session.qr"        →  sends QR string
4. Your app  →  renders QR code           →  client scans with phone
5. OpenWA    →  emits "session.ready"     →  session is active
```

## Prerequisites

- OpenWA running and accessible
- A valid API key (`X-API-Key`)
- npm packages: `socket.io-client`, `qrcode`

```bash
npm install socket.io-client qrcode
```

## API Reference

### Create Session

```http
POST /api/sessions
X-API-Key: your-api-key
Content-Type: application/json

{
  "name": "client-123"
}
```

**Response:**
```json
{
  "id": "35c7271e-44bb-4d6f-ac6c-f24359c745d0",
  "name": "client-123",
  "status": "initializing"
}
```

### WebSocket Events

| Event | Direction | Payload |
|-------|-----------|---------|
| `subscribe` | Client → Server | `{ sessionId }` |
| `session.qr` | Server → Client | `{ sessionId, qr }` |
| `session.ready` | Server → Client | `{ sessionId, phone, pushName }` |
| `session.disconnected` | Server → Client | `{ sessionId }` |

## JavaScript Integration

```js
import { io } from 'socket.io-client'
import QRCode from 'qrcode'

const API_URL = 'https://api.smartcode-labs.com'
const API_KEY = 'your-api-key'

// Step 1: Create session
async function createSession(name) {
  const res = await fetch(`${API_URL}/api/sessions`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': API_KEY,
    },
    body: JSON.stringify({ name }),
  })
  return res.json()
}

// Step 2: Connect WebSocket and listen for QR
function connectSession(sessionId, { onQR, onReady, onDisconnected }) {
  const socket = io(API_URL, {
    extraHeaders: { 'X-API-Key': API_KEY },
  })

  socket.on('connect', () => {
    socket.emit('subscribe', { sessionId })
  })

  socket.on('session.qr', async ({ qr }) => {
    const dataUrl = await QRCode.toDataURL(qr)
    onQR?.(dataUrl)
  })

  socket.on('session.ready', (data) => {
    onReady?.(data)
  })

  socket.on('session.disconnected', (data) => {
    onDisconnected?.(data)
  })

  return socket
}
```

## React Component

```tsx
import { useEffect, useState } from 'react'
import { createSession, connectSession } from './whatsapp'

interface Props {
  clientId: string
  onConnected: (phone: string) => void
}

export function WhatsAppQR({ clientId, onConnected }: Props) {
  const [qrUrl, setQrUrl] = useState<string | null>(null)
  const [status, setStatus] = useState<'loading' | 'qr' | 'ready'>('loading')

  useEffect(() => {
    let socket: ReturnType<typeof connectSession>

    createSession(`client-${clientId}`).then(({ id }) => {
      socket = connectSession(id, {
        onQR: (url) => {
          setQrUrl(url)
          setStatus('qr')
        },
        onReady: ({ phone }) => {
          setStatus('ready')
          onConnected(phone)
        },
        onDisconnected: () => {
          setStatus('loading')
          setQrUrl(null)
        },
      })
    })

    return () => {
      socket?.disconnect()
    }
  }, [clientId])

  if (status === 'ready') {
    return <p>WhatsApp connected successfully.</p>
  }

  if (status === 'qr' && qrUrl) {
    return (
      <div>
        <p>Scan this QR code with your WhatsApp app.</p>
        <img src={qrUrl} alt="WhatsApp QR Code" width={256} height={256} />
      </div>
    )
  }

  return <p>Generating QR code...</p>
}
```

## Phone Number Format

When sending messages, use the following format:

```
[country_code][number]@c.us
```

| Country | Code | Example |
|---------|------|---------|
| Mexico | 521 | `5215512345678@c.us` |
| Argentina | 54 | `5491112345678@c.us` |
| Spain | 34 | `34612345678@c.us` |
| USA | 1 | `12025551234@c.us` |
| Colombia | 57 | `573001234567@c.us` |

> **Note:** Mexico requires an extra `1` after the country code `52` for mobile numbers.

## Send a Message

Once the session is ready:

```js
async function sendMessage(sessionId, chatId, text) {
  const res = await fetch(`${API_URL}/api/sessions/${sessionId}/messages/send-text`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-API-Key': API_KEY,
    },
    body: JSON.stringify({ chatId, text }),
  })
  return res.json()
}

// Usage
await sendMessage(
  '35c7271e-44bb-4d6f-ac6c-f24359c745d0',
  '5215512345678@c.us',
  'Hello from OpenWA!'
)
```

## Session Lifecycle

```
initializing → qr_ready → connecting → ready → disconnected
                  ↑                               |
                  └───────────────────────────────┘
                         (re-scan required)
```

## Security Considerations

- Never expose your `API_KEY` in frontend code. Route requests through your own backend.
- Each client should have their own isolated session.
- Store the `sessionId` associated with each client in your database.

---

<div align="center">

[← 22 - n8n Integration](./22-n8n-integration.md) · [Documentation Index](./README.md)

</div>
