---
name: new-service
description: Scaffold service + Zustand store untuk domain baru tanpa CRUD penuh. Argumen: <domain> <endpoint-prefix> <folder>
---

# New Service + Store Scaffold

Buatkan service dan store baru untuk domain: **$ARGUMENTS**

Format argumen: `<domain> <endpoint-prefix> <folder>`
Contoh: `mqtt-user /rabbitmq-management/user settings`

---

## 1. Type — `src/types/data/<folder>/<domain>.ts`
Buat interface `I<Domain>` dengan field yang relevan.

## 2. Store — `src/store/<folder>/<domain>.state.ts`
Buat Zustand store dengan pattern batch setter:
```ts
import { create } from 'zustand'

interface I<Domain>Store {
  data: I<Domain>[]
  totalData: number
  sidebarOpen: boolean
  id: number | null
  search: string
  limit: number
  offset: number
  setData: (data: I<Domain>[], totalData: number) => void
  setSearch: (search: string) => void
  setLimit: (limit: number) => void
  setOffset: (offset: number) => void
  setSidebarOpen: (open: boolean) => void
  setId: (id: number | null) => void
}

export const use<Domain>Store = create<I<Domain>Store>((set) => ({
  data: [],
  totalData: 0,
  sidebarOpen: false,
  id: null,
  search: '',
  limit: 10,
  offset: 0,
  setData: (data, totalData) => set({ data, totalData }),
  setSearch: search => set({ search }),
  setLimit: limit => set({ limit }),
  setOffset: offset => set({ offset }),
  setSidebarOpen: sidebarOpen => set({ sidebarOpen }),
  setId: id => set({ id })
}))
```

## 3. Service — `src/services/<folder>/<domain>.service.ts`
Buat fungsi CRUD menggunakan `http` dari `src/configs/http` dan endpoint dari `src/configs/api`.

Pattern getList:
```ts
export const get<Domain>s = async () => {
  const state = use<Domain>Store.getState()
  const response = await http.get(api.<domain>.list, {
    params: { limit: state.limit, offset: state.offset, ...(state.search ? { search: state.search } : {}) }
  })
  use<Domain>Store.getState().setData(response.data.data.list, response.data.data.total)
}
```

## 4. API Endpoint — `src/configs/api.ts`
Tambahkan endpoint di lokasi yang sesuai dalam object `api`.

---

Setelah selesai, tampilkan ringkasan file yang dibuat.

$ARGUMENTS
