---
name: socketio
description: Referensi lengkap penggunaan Socket.IO di project ini — dua pola utama (React component & custom hook), dua useEffect pattern (connect vs joinRoom), auth token, room management, reconnect handling, dan semua socket path aktif. Gunakan saat membuat fitur live/realtime baru, debug koneksi socket, atau menambah room baru.
---

# Socket.IO — Realtime Reference

---

## Socket yang Aktif di Project

| Socket | Path | File | Tujuan |
|--------|------|------|--------|
| Client Dashboard | `/socket/dashboard-client` | `LiveClientDashboard.tsx` | Realtime transaksi per merchant |
| Admin Dashboard | `/socket/dashboard-admin` | `LiveAdminDashboard.tsx` | Realtime dashboard admin/merchant |
| Gate RFID | `/websocket/rfid` | `liveGate.tsx` | Realtime scan RFID gate |
| Plantation | `/simulation/plantation/` | `usePlantationSocket.ts` | GPS & simulasi perkebunan |

**URL WebSocket:** selalu ambil dari `config.ws` — hasil dari `src/configs/http.ts` (auto-generate dari subdomain di production, atau dari `NEXT_PUBLIC_WS_URL` di dev/staging).

```ts
import { config } from 'src/configs/http'
const SOCKET_URL = config.ws
```

---

## Pola 1 — React Component (renders null)

Dipakai oleh: `LiveClientDashboard`, `LiveAdminDashboard`, `LiveGate`

**Struktur wajib: dua `useEffect` terpisah.**

```tsx
import React, { useEffect, useRef } from 'react'
import io, { Socket } from 'socket.io-client'
import { config } from 'src/configs/http'
import { useMyFeatureStore } from 'src/store/my-feature.state'

type Props = { someParam: string }

const LiveMyFeature: React.FC<Props> = ({ someParam }) => {
  const socketRef = useRef<Socket | null>(null)
  const previousRoomRef = useRef<any>(null)
  const isMountedRef = useRef<boolean>(true)
  const isJoiningRef = useRef<boolean>(false)

  // useEffect #1 — Connect/disconnect (jalankan sekali, cleanup saat unmount)
  useEffect(() => {
    isMountedRef.current = true

    const socket = io(config.ws, {
      transports: ['websocket'],
      path: '/socket/my-feature'
    })
    socketRef.current = socket

    socket.on('connect', () => {
      const token = localStorage.getItem('accessToken') || ''
      socket.emit('auth', { auth: token })
    })

    socket.on('message', (parsedData: any) => {
      if (!isMountedRef.current) return
      try {
        const data = typeof parsedData === 'string' ? JSON.parse(parsedData) : parsedData
        if (data) {
          useMyFeatureStore.getState().setSocketData(data)
        }
      } catch (error) {
        console.error('Error parsing socket message:', error)
      }
    })

    socket.on('reconnect', () => {
      // Auto-rejoin room setelah reconnect
      if (previousRoomRef.current) {
        const token = localStorage.getItem('accessToken') || ''
        socket.emit('joinRoom', JSON.stringify({ ...previousRoomRef.current, auth: token }))
      }
    })

    socket.on('disconnect', (reason: string) => {
      if (!isMountedRef.current || reason === 'io client disconnect') return
      useMyFeatureStore.getState().setLoading(true)
    })

    socket.on('connect_error', e => console.error('Socket error:', e))
    socket.on('reconnect_failed', () => useMyFeatureStore.getState().setLoading(false))

    return () => {
      isMountedRef.current = false
      if (socket.connected) socket.disconnect()
    }
  }, []) // deps: kosong atau [url] — JANGAN masukkan state

  // useEffect #2 — Join/leave room (jalankan saat param berubah)
  useEffect(() => {
    const socket = socketRef.current
    if (!socket || !someParam) return

    const joinRoom = () => {
      if (isJoiningRef.current) return
      isJoiningRef.current = true

      const token = localStorage.getItem('accessToken') || ''
      const joinData = { roomId: someParam, auth: token }

      // Leave room sebelumnya dulu
      if (previousRoomRef.current) {
        socket.emit('leaveRoom', JSON.stringify(previousRoomRef.current))
        previousRoomRef.current = null
      }

      socket.emit('joinRoom', JSON.stringify(joinData))
      previousRoomRef.current = joinData

      setTimeout(() => { isJoiningRef.current = false }, 1000)
    }

    if (socket.connected) {
      joinRoom()
    } else {
      socket.once('connect', joinRoom)
    }

    return () => {
      if (socket.connected && previousRoomRef.current) {
        socket.emit('leaveRoom', JSON.stringify(previousRoomRef.current))
        previousRoomRef.current = null
      }
    }
  }, [someParam]) // deps: parameter yang menentukan room

  return null
}

export default LiveMyFeature
```

---

## Pola 2 — Custom Hook

Dipakai oleh: `usePlantationSocket`  
Cocok ketika komponen butuh data langsung dari socket (bukan hanya via store).

```ts
import { useState, useEffect, useRef } from 'react'
import io, { Socket } from 'socket.io-client'
import { config } from 'src/configs/http'

export const useMySocket = (roomId?: string) => {
  const [data, setData] = useState<any>(null)
  const [isConnected, setIsConnected] = useState(false)
  const socketRef = useRef<Socket | null>(null)
  const isMountedRef = useRef(true)

  // useEffect #1: connect/disconnect
  useEffect(() => {
    isMountedRef.current = true

    const socket = io(config.ws, {
      transports: ['websocket'],
      path: '/socket/my-feature'
    })
    socketRef.current = socket

    socket.on('connect', () => {
      if (!isMountedRef.current) return
      setIsConnected(true)
      // Auto-join jika ada roomId
      if (roomId) socket.emit('join_room', roomId)
    })

    socket.on('my_event', (incoming: any) => {
      if (!isMountedRef.current) return
      setData(incoming)
    })

    socket.on('disconnect', (reason) => {
      if (!isMountedRef.current || reason === 'io client disconnect') return
      setIsConnected(false)
    })

    socket.on('connect_error', e => console.error('Socket error:', e))

    return () => {
      isMountedRef.current = false
      if (socket.connected) socket.disconnect()
    }
  }, []) // hanya sekali

  // useEffect #2: join/leave room saat roomId berubah
  useEffect(() => {
    const socket = socketRef.current
    if (!socket || !socket.connected || !roomId) return

    socket.emit('join_room', roomId)

    return () => {
      socket.emit('leave_room', roomId)
    }
  }, [roomId])

  return { data, isConnected }
}
```

---

## Pola 3 — One-Shot (saat Login)

Dipakai di `AuthContext` — connect, ambil data awal, langsung disconnect.

```ts
const socket = io(config.ws, {
  transports: ['websocket'],
  path: '/socket/dashboard-client'
})

socket.on('connect', () => {
  socket.emit('auth', { auth: token })
  socket.emit('joinRoom', JSON.stringify({ companyId: 0, branchId: 0, merchantId: [0], auth: token }))
})

socket.on('message', (parsedData: any) => {
  try {
    const data = typeof parsedData === 'string' ? JSON.parse(parsedData) : parsedData
    if (data) useMyStore.getState().setSocketData(data)
  } catch (_) {}
  socket.disconnect() // disconnect setelah terima data pertama
})

socket.on('connect_error', () => socket.disconnect())
```

---

## Auth Token

Selalu emit `auth` segera setelah `connect`, sebelum emit `joinRoom`:

```ts
socket.on('connect', () => {
  const token = localStorage.getItem('accessToken') || ''
  socket.emit('auth', { auth: token })
  // emit joinRoom setelah ini (atau dengan delay 500ms jika perlu)
})
```

---

## Room Management

### Client Dashboard
```ts
// Join
socket.emit('joinRoom', JSON.stringify({
  companyId: 0,
  branchId: 0,
  merchantId: [merchantId],  // array of number
  auth: token,
  search: '',
  limit: 999,
  offset: 0,
  date: '2025-01-01'
}))

// Leave
socket.emit('leaveRoom', JSON.stringify(previousRoomData))
```

### Admin Dashboard
```ts
// Join
socket.emit('adminJoinRoom', JSON.stringify({
  companyId: 0,
  branchId: 0,
  merchantId: [merchantId],
  auth: token
}))

// Leave
socket.emit('adminLeaveRoom', JSON.stringify(previousRoomData))
```

### Gate RFID
```ts
socket.emit('auth', { tokenRequest: token })
socket.emit('joinRoom', JSON.stringify({ id, categoryId }))
socket.emit('leaveRoom', JSON.stringify({ id, categoryId }))
```

### Plantation GPS
```ts
socket.emit('join_room', deviceName)   // string, bukan JSON
socket.emit('leave_room', deviceName)
```

---

## Merchant Selection — Hierarki Room

| Seleksi | companyId | branchId | merchantId |
|---------|-----------|----------|------------|
| Company | X | 0 | [0] |
| Branch | 0 | X | [semua merchant ID di branch] |
| Merchant tunggal | 0 | 0 | [X] |

**Filter data masuk** — hanya filter jika merchant tunggal dipilih (`id > 0`):
```ts
const currentState = useTransactionClientStore.getState()
if (currentState.id > 0) {
  const incomingId = data.merchant?.id != null ? Number(data.merchant.id) : null
  if (incomingId !== null && incomingId !== currentState.id) return // abaikan
}
```

---

## Parse Data Socket

Server bisa mengirim string JSON atau object langsung. Selalu handle keduanya:

```ts
socket.on('message', (parsedData: any) => {
  const data = typeof parsedData === 'string' ? JSON.parse(parsedData) : parsedData
  // ...
})
```

---

## Ref-ref Penting

| Ref | Tujuan |
|-----|--------|
| `socketRef` | Simpan instance socket agar bisa diakses dari useEffect #2 |
| `previousRoomRef` | Simpan data room terakhir untuk leaveRoom dan auto-rejoin |
| `isMountedRef` | Cegah setState setelah component unmount |
| `isJoiningRef` | Cegah double-join saat parameter berubah cepat |

---

## Aturan Penting

| Situasi | Solusi |
|---------|--------|
| Jangan taruh `state` di deps useEffect #1 | Zustand store stabil, tapi memasukkannya menyebabkan reconnect loop |
| Akses store di dalam socket handler | `useXxxStore.getState()` — bukan hook |
| Join room setelah connect | Cek `socket.connected` dulu, kalau belum pakai `socket.once('connect', joinRoom)` |
| Auto-rejoin setelah reconnect | Simpan data room di `previousRoomRef`, emit ulang saat event `reconnect` atau `connect` |
| Cleanup useEffect #2 | Emit `leaveRoom` saja — JANGAN disconnect socket di sini |
| Cleanup useEffect #1 | `socket.disconnect()` saat unmount |
| Socket init saat login | Cek permission menu dulu (`hasClient`, `hasDashboardMerchant`) sebelum connect |

---

## Events yang Dihandle

```ts
socket.on('connect', handler)         // berhasil connect / reconnect
socket.on('disconnect', handler)      // terputus (reason: string)
socket.on('message', handler)         // data realtime dari server
socket.on('reconnect', handler)       // berhasil reconnect (setelah putus)
socket.on('reconnect_attempt', handler) // sedang mencoba reconnect
socket.on('reconnect_error', handler) // gagal satu percobaan reconnect
socket.on('reconnect_failed', handler) // semua percobaan gagal
socket.on('connect_error', handler)   // gagal connect awal
socket.on('error', handler)           // error umum socket
```

$ARGUMENTS
