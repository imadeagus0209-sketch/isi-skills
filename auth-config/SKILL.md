---
name: auth-config
description: Referensi lengkap arsitektur Auth — AuthContext, useAuth hook, http interceptor, socket init, dan provider tree di _app.tsx. Gunakan saat mengubah login/logout flow, menambah field user, mengubah token handling, atau debug auth issue.
---

# Auth & Config Architecture Reference

Panduan arsitektur auth project ini. Baca sebelum menyentuh login/logout, token, socket init, atau provider tree.

---

## 1. File Map

| File | Peran |
|------|-------|
| `src/configs/auth.ts` | Konstanta endpoint & key localStorage |
| `src/configs/http.ts` | Axios instance + URL resolver + interceptors |
| `src/context/AuthContext.tsx` | AuthProvider: login, logout, socket init |
| `src/context/types.ts` | Type: `UserDataType`, `AuthValuesType`, `ErrCallbackType` |
| `src/hooks/useAuth.tsx` | `useAuth()` — shortcut ke `useContext(AuthContext)` |
| `src/pages/_app.tsx` | Provider tree, Guard, route-change side effects |

---

## 2. Auth Config (`src/configs/auth.ts`)

```ts
export default {
  loginEndpoint: api.login,
  logoutEnpoint: api.logout,   // ← typo intentional, jangan ubah
  storageTokenKeyName: 'accessToken'
}
```

**Penting:** Selalu pakai `authConfig.storageTokenKeyName`, bukan string literal `'accessToken'`.

---

## 3. HTTP Client (`src/configs/http.ts`)

### URL Resolution
- **Dev/Staging:** baca dari `NEXT_PUBLIC_API_URL` dan `NEXT_PUBLIC_WS_URL`
- **Production:** URL di-generate otomatis dari `window.location.hostname` via fungsi `gen()`
- Export `config.api` (base URL) dan `config.ws` (WebSocket URL)

### Interceptor Request
Auto-attach `Authorization: Bearer <token>` dari `localStorage.accessToken` ke setiap request.

### Interceptor Response
Status dibaca dari `response.data.headers.statusCode` (bukan HTTP status):

| statusCode | Aksi |
|-----------|------|
| `400` | reject promise |
| `403` | tampilkan snackbar error + reject |
| `401` | logout → clear localStorage → redirect `/login/` |
| `500` | throw reject |

HTTP 401 dari network layer juga ditangani dengan aksi yang sama.

---

## 4. AuthContext (`src/context/AuthContext.tsx`)

### State
```ts
user: UserDataType | null   // { name, role, id, idRole }
loading: boolean
```

### `handleLogin(params, errorCallback?)`
1. `POST /auth/login/v1` → dapat token + user data
2. Simpan token ke `localStorage.accessToken`
3. Simpan `userData` ke `localStorage.userData`
4. Fetch `/admin/access-menu` → dapat daftar menu
5. `initSocketsOnLogin(token, menus)` — init socket sesuai akses menu
6. Jika `first_login_token` ada → redirect `/change-password`
7. Jika tidak → redirect ke `returnUrl` atau `/`

### `handleLogout()`
1. `POST /auth/logout/v1`
2. `setUser(null)` + clear localStorage
3. Reset store: `useAdminStore`, `useTransactionClientStore`, `useTransactionAdminStore`
4. `router.push('/login/')`

### `initSocketsOnLogin(token, menus)`
Init socket hanya jika user punya akses menu:
- Menu `'Client'` → socket ke `/socket/dashboard-client`
- Menu `'Dashboard Merchant'` → socket ke `/socket/dashboard-admin`

### Menambah field ke `UserDataType`
1. Tambah field di `src/context/types.ts`
2. Tambah field di `handleLogin` saat bentuk object `userData`
3. Pastikan field ada di response `IResponseLogin`

---

## 5. useAuth Hook

```ts
import { useAuth } from 'src/hooks/useAuth'

const { user, login, logout, loading } = useAuth()
```

---

## 6. Provider Tree (`src/pages/_app.tsx`)

```
CacheProvider (Emotion)
└── AuthProvider
    └── SettingsProvider
        └── ThemeComponent
            └── WindowWrapper
                └── Guard (AuthGuard | GuestGuard | none)
                    └── AclGuard
                        └── Component (halaman)
```

### Guard Logic
```ts
if (guestGuard)       → GuestGuard   // halaman login, forgot-password
else if (!authGuard)  → tanpa guard  // halaman public
else                  → AuthGuard    // semua halaman lain
```

Set guard di halaman:
```ts
PageName.guestGuard = true    // guest only
PageName.authGuard = false    // public
```

---

## 7. Pola Umum

```ts
// Akses user
const { user } = useAuth()

// Logout dari mana saja
const { logout } = useAuth()

// Cek token di service
const token = localStorage.getItem('accessToken')

// Tambah socket baru saat login — di initSocketsOnLogin()
const hasFeature = menus.some(m => m.name === 'Nama Menu')
if (hasFeature) {
  const socket = io(config.ws, { transports: ['websocket'], path: '/socket/path-baru' })
}
```

$ARGUMENTS
