---
name: axios-http
description: Referensi penggunaan Axios di project ini — instance http, pola GET/POST/PUT/DELETE di service, cara baca response IResponse<T>, error handling, dan kapan pakai http vs axios mentah. Gunakan saat membuat service baru, debug API call, atau menambah endpoint.
---

# Axios & HTTP — Service Layer Reference

---

## Instance Axios

Project menggunakan satu instance axios global:

```ts
import http from 'src/configs/http'
import { api } from 'src/configs/api'
```

**Jangan** import `axios` langsung untuk call API internal — selalu pakai `http`.  
`axios` mentah hanya dipakai untuk endpoint luar (contoh: `aktivasi-edc.service.ts`).

---

## Apa yang `http` lakukan secara otomatis

| Fitur | Detail |
|-------|--------|
| Base URL | Auto-generate dari subdomain (production) atau env `NEXT_PUBLIC_API_URL` (dev/staging) |
| Auth Header | Auto-attach `Authorization: Bearer <accessToken>` dari `localStorage` |
| CSRF refresh | Baca header `csrf` dari response, simpan ke localStorage |
| statusCode 400/403/500 | Reject otomatis dari response interceptor |
| statusCode 401 | Auto logout + redirect ke `/login` |

---

## Tambah Endpoint Baru

**Selalu** tambah ke `src/configs/api.ts`, jangan hardcode URL di service:

```ts
// src/configs/api.ts
export const api = {
  // ... existing
  myFeature: {
    getList: '/my-feature/get-list',
    getItem: '/my-feature/get-item',
    create: '/my-feature/create',
    update: '/my-feature/update',
    delete: '/my-feature/delete'
  }
}
```

---

## Pola GET dengan Query Params (dari Store)

```ts
import { AxiosResponse } from 'axios'
import { api } from 'src/configs/api'
import http from 'src/configs/http'
import { useMyFeatureStore } from 'src/store/my-feature.state'
import { IResponse } from 'src/types/data/response'
import { IMyFeature } from 'src/types/data/my-feature'

export const getMyFeatureList = async () => {
  const store = useMyFeatureStore.getState()

  try {
    const response: AxiosResponse<IResponse<{ listMyFeature: IMyFeature[]; totalData: number }>> =
      await http.get(api.myFeature.getList, {
        params: {
          search: store.search,
          limit: store.limit,
          offset: store.offset
        }
      })

    store.setData(response.data.data.listMyFeature, response.data.data.totalData)

    return response.data
  } catch (error) {
    console.error('Error fetching:', error)
  }
}
```

---

## Pola GET by ID

```ts
export const getMyFeatureById = async (id: number) => {
  try {
    const response: AxiosResponse<IResponse<IMyFeature>> =
      await http.get(api.myFeature.getItem, { params: { id } })

    return response.data.data
  } catch (error) {
    console.error('Error fetching by id:', error)
    throw error
  }
}
```

---

## Pola POST (Create)

```ts
import { useSnackbarStore } from 'src/store/snackbar.state'

export const createMyFeature = async (payload: ICreateMyFeature) => {
  try {
    const response = await http.post(api.myFeature.create, payload)

    useSnackbarStore.getState().showMessage({
      message: response.data.headers?.message || 'Created successfully',
      severity: 'success',
      key: Date.now()
    })

    return response.data
  } catch (error: any) {
    useSnackbarStore.getState().showMessage({
      message: error?.data?.headers?.message || 'Failed to create',
      severity: 'error',
      key: Date.now()
    })
    throw error
  }
}
```

---

## Pola PUT (Update)

```ts
export const updateMyFeature = async (payload: IUpdateMyFeature) => {
  try {
    const response = await http.put(api.myFeature.update, payload)

    useSnackbarStore.getState().showMessage({
      message: response.data.headers?.message || 'Updated successfully',
      severity: 'success',
      key: Date.now()
    })

    return response.data
  } catch (error: any) {
    useSnackbarStore.getState().showMessage({
      message: error?.data?.headers?.message || 'Failed to update',
      severity: 'error',
      key: Date.now()
    })
    throw error
  }
}
```

---

## Pola DELETE

```ts
export const deleteMyFeature = async (id: number) => {
  try {
    const response = await http.delete(api.myFeature.delete, { params: { id } })

    useSnackbarStore.getState().showMessage({
      message: 'Deleted successfully',
      severity: 'success',
      key: Date.now()
    })

    return response.data
  } catch (error: any) {
    useSnackbarStore.getState().showMessage({
      message: error?.data?.headers?.message || 'Failed to delete',
      severity: 'error',
      key: Date.now()
    })
    throw error
  }
}
```

---

## Pola Upload File (FormData)

```ts
export const uploadFile = async (file: File) => {
  const formData = new FormData()
  formData.append('file', file)

  const response = await http.post(api.uploadFile, formData, {
    headers: { 'Content-Type': 'multipart/form-data' }
  })

  return response.data
}
```

---

## IResponse — Struktur Response API

```ts
// src/types/data/response.ts
interface IResponse<T> {
  data: T
  headers: {
    statusCode: number
    message: string
  }
}
```

Akses data: `response.data.data` (bukan `response.data`).  
Akses message: `response.data.headers.message`.

---

## Axios Mentah (Endpoint Eksternal)

Hanya untuk layanan luar yang tidak butuh token internal (contoh: EDC activation service):

```ts
import axios from 'axios'

const edcHttp = axios.create({
  baseURL: 'http://192.168.3.25:30058',
  headers: { 'Content-Type': 'application/json' }
})

export const activateEdc = async ({ otp, apiKey }: IActivateEdcPayload) => {
  const response = await edcHttp.post(
    '/edc/activate',
    { otp },
    { headers: { 'x-api-key': apiKey } }
  )
  return response.data
}
```

---

## Error Handling

Response interceptor di `http.ts` auto-reject statusCode 400/401/403/500.  
`catch (error: any)` menerima object `AxiosResponse` asli (bukan `AxiosError`) karena interceptor me-reject `response` bukan `error`.

```ts
try {
  await http.post(api.myFeature.create, payload)
} catch (error: any) {
  // error di sini adalah AxiosResponse yang di-reject interceptor
  const message = error?.data?.headers?.message || 'Something went wrong'
  console.error(message)
}
```

---

## Kapan Panggil Service

Service dipanggil dari komponen atau handler, **bukan** langsung dari store:

```ts
// Di komponen — saat mount atau refresh
useEffect(() => {
  getMyFeatureList()
}, [store.search, store.limit, store.offset])

// Saat submit form
const onSubmit = async (data: ICreateMyFeature) => {
  await createMyFeature(data)
  await getMyFeatureList()  // refresh list
  setDialogOpen(false)
}
```

---

## Aturan

| Situasi | Solusi |
|---------|--------|
| API internal project | Pakai `http` dari `src/configs/http` |
| Endpoint luar / tidak butuh token | Buat instance `axios.create()` sendiri |
| Tambah endpoint baru | Daftarkan di `src/configs/api.ts` dulu |
| Baca query params dari store | `useXxxStore.getState()` di dalam service |
| Notifikasi sukses/gagal | `useSnackbarStore.getState().showMessage(...)` |
| Akses data response | `response.data.data` (nested dua kali) |

$ARGUMENTS
