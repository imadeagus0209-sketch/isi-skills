---
name: http-config
description: Referensi konfigurasi HTTP dinamis — cara URL API dan WebSocket di-generate otomatis dari subdomain/domain di production, dan dari env var di dev/staging. Gunakan saat menambah domain baru, debug URL salah, atau mengubah pola subdomain API.
---

# HTTP Config — Dynamic URL Resolution

File: `src/configs/http.ts`

---

## Konsep Utama

Di **production**, URL API dan WebSocket **tidak pakai env var** — di-generate otomatis dari `window.location.hostname` menggunakan logika transformasi subdomain.

Di **dev/staging**, URL diambil dari env var `NEXT_PUBLIC_API_URL` dan `NEXT_PUBLIC_WS_URL`.

---

## Daftar Domain & Subdomain yang Dikenal

### Subdomain Admin (hostsplit[0])

| Subdomain | Jenis | Keterangan |
|-----------|-------|------------|
| `<app>admin` | Production | Admin standard |
| `<app>adminlab` | Lab / Staging | 3 huruf terakhir: `lab` |
| `<app>adminuat` | UAT | 3 huruf terakhir: `uat` |
| `<app>ecu` | Production | Variant ECU |
| `demo<app>` | Production | Demo standard |
| `demo<app>ecu` | Production | Demo ECU |

### Subdomain API yang Di-generate

| Admin subdomain | → | API subdomain |
|----------------|---|---------------|
| `<app>admin` | → | `<app>api` |
| `<app>adminlab` | → | `<app>apilab` |
| `<app>adminuat` | → | `<app>apiuat` |
| `<app>ecu` | → | `<app>api` |
| `demo<app>` | → | `demoapi` |
| `demo<app>ecu` | → | `demoapi` |

> `<app>` = nama aplikasi yang di-replace di fungsi `gen()` dalam `src/configs/http.ts`

### Domain Utama (hostsplit[1])

Setiap client menggunakan domain miliknya sendiri, contoh:
- `example.com`
- `example.co.id`
- `example.id`

Format URL API final: `<app>api.<domain-client>`

---

## Cara Kerja Step-by-Step

### Step 1 — Split hostname
```ts
const hostsplit = hostname.split(/\.([^\n]*)/)
// "appadmin.example.id" → ["appadmin", "example.id", ""]
// hostsplit[0] = subdomain   → "appadmin"
// hostsplit[1] = sisa domain → "example.id"
```

### Step 2 — Fungsi `gen(subdomain, suffix)`
Mengubah subdomain **admin** menjadi subdomain **API**:

```ts
const gen = (hostsplit: string, reg: string): string => {
  const slice_3 = hostsplit.slice(hostsplit.length - 3)        // 3 char terakhir
  const environment = slice_3 === 'lab' || slice_3 === 'uat'  // cek env staging/uat

  const sub_domain = hostsplit
    .replace('<app>admin', '<app>')
    .replace('<app>ecu',   '<app>')
    .replace('demo<app>',  'demo')
    .replace('demo<app>ecu', 'demo')

  // Production:  sub_domain + 'api'                        → "<app>api"
  // Lab/UAT:     sub_domain (tanpa suffix) + 'api' + suffix → "<app>apilab"
  return environment
    ? sub_domain.replace(slice_3, '') + reg + slice_3
    : sub_domain + reg
}
```

### Step 3 — Contoh transformasi

| Hostname | hostsplit[0] | gen() result | apiname |
|----------|-------------|-------------|---------|
| `appadmin.example.id` | `appadmin` | `appapi` | `appapi.example.id` |
| `appadminlab.example.id` | `appadminlab` | `appapilab` | `appapilab.example.id` |
| `appadminuat.example.id` | `appadminuat` | `appapiuat` | `appapiuat.example.id` |
| `demoadmin.example.id` | `demoadmin` | `demoapi` | `demoapi.example.id` |
| `appecu.example.com` | `appecu` | `appapi` | `appapi.example.com` |

### Step 4 — Pilih env vs production
```ts
const apiname = process.env.NODE_ENV === 'production'
  ? `${apiregex}.${hostsplit[1]}`  // dari hostname parsing
  : hostApi.API                     // dari NEXT_PUBLIC_API_URL

const wsname = process.env.NODE_ENV === 'production'
  ? `${wsregex}.${hostsplit[1]}`
  : hostApi.WS                      // dari NEXT_PUBLIC_WS_URL
```

### Step 5 — Build URL final
```ts
const api = hostsplit[0].search('html') >= 0
  ? `${protocol}//${apiname}:80`
  : `${protocol}//${apiname}`

const wsRaw = hostsplit[0].search('html') >= 0
  ? `${protocol.replace('http', 'ws')}//${apiname}:7999`
  : `${protocol.replace('http', 'ws')}//${wsname}`

// Fallback ke env var jika parsing gagal (SSR / hostname kosong)
const ws = wsRaw.includes('undefined') || wsRaw === 'wss://' || wsRaw === 'ws://'
  ? hostApi.WS || ''
  : wsRaw
```

---

## Export yang Tersedia

```ts
import http, { config } from 'src/configs/http'

config.api   // base URL API
config.ws    // WebSocket URL
http         // Axios instance (sudah ada interceptor)
```

---

## Axios Instance & Interceptors

### Cara Pakai di Service
```ts
import http from 'src/configs/http'
import { api } from 'src/configs/api'

const response = await http.get(api.master.bank.get, { params: { limit, offset } })
const response = await http.post(api.master.bank.create, payload)
const response = await http.put(api.master.bank.update, payload)
const response = await http.delete(api.master.bank.delete, { params: { id } })
```

Tidak perlu attach token manual — sudah ditangani interceptor.

### Interceptor Request — Auto Bearer Token
Setiap request otomatis ditambahkan `Authorization: Bearer <token>` dari `localStorage.accessToken`.

### Interceptor Response — Status dari Body

Status dibaca dari `response.data.headers.statusCode` (bukan HTTP status):

| statusCode | Aksi Interceptor |
|-----------|-----------------|
| `200` | Lolos normal |
| `400` | reject promise |
| `403` | Snackbar error otomatis + reject |
| `401` | Logout → clear localStorage → redirect `/login/` |
| `500` | throw reject |

**CSRF Token:** Jika response header mengandung `csrf`, token baru otomatis disimpan ke `localStorage.accessToken`.

### HTTP Network Error 401
```
error.response?.status === 401 → Logout → localStorage.clear() → redirect /login/
```

### Pola Error Handling di Service
```ts
export const createItem = async (payload: ICreate) => {
  try {
    const response = await http.post(api.item.create, payload)
    return response.data
  } catch (error: any) {
    throw error  // error.data.headers.message → pesan dari server
  }
}

// Di komponen
try {
  await createItem(data)
  snackbar.showMessage({ message: 'Berhasil', severity: 'success', key: Date.now() })
} catch (error: any) {
  snackbar.showMessage({
    message: error?.data?.headers?.message || 'Gagal',
    severity: 'error',
    key: Date.now()
  })
}
```

### Yang TIDAK Perlu Dilakukan
- Jangan cek HTTP 401/403 manual → sudah auto logout/snackbar
- Jangan attach `Authorization` header manual → sudah auto
- Jangan refresh token manual → sudah auto dari CSRF header

---

## Menambah Subdomain Baru

Tambahkan `.replace()` di fungsi `gen()` dalam `src/configs/http.ts`:

```ts
const sub_domain = hostsplit
  .replace('<app>admin', '<app>')
  .replace('<app>ecu',   '<app>')
  .replace('demo<app>',  'demo')
  .replace('<newapp>admin', '<newapp>')   // ← tambahkan di sini
```

---

## Mode IP (Saat Ini Di-comment)

```ts
const apiname = process.env.NODE_ENV === 'production' ? `${hostname}:${portapi}` : hostApi.API
const wsname  = process.env.NODE_ENV === 'production' ? `${hostname}:${portws}`  : hostApi.WS
const portapi = 30030
const portws  = 30030
```

---

## Checklist Debug URL Salah

- [ ] `hostsplit[0]` — subdomain admin cocok dengan `.replace()` di `gen()`?
- [ ] `hostsplit[1]` — domain utama sudah benar?
- [ ] 3 huruf terakhir subdomain: `lab` atau `uat`?
- [ ] DevTools Console → `window.location.hostname` untuk verifikasi

$ARGUMENTS
