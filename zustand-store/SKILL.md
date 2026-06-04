---
name: zustand-store
description: Referensi lengkap penggunaan Zustand store sebagai global state — pola store, cara akses di komponen vs service/socket, store global project (snackbar, dialog, accessMenu), naming convention, dan direktori. Gunakan saat membuat store baru, mengakses state dari service, atau debug state tidak update.
---

# Zustand Store — Global State Reference

---

## Direktori Store

```
src/store/
├── snackbar.state.ts          # global — notifikasi
├── dialog.state.ts            # global — confirm dialog
├── admin.state.ts             # global — accessMenu (permission)
├── master/                    # CRUD master data
├── settings/                  # halaman settings
├── transaction/               # dashboard & transaksi
├── plantation/                # fitur perkebunan
├── gate/                      # live gate
├── dashboard/                 # dashboard
├── lorawan/                   # LoRaWAN device/gateway
├── manageRoles/               # kelola role & menu
└── storege/                   # warehouse/pallet/product
```

Naming: `use<Domain>Store` → file `<domain>.state.ts`

---

## Pola 1 — Basic CRUD Store

```ts
import { create } from 'zustand'
import { IBank } from 'src/types/data/master/bank'

interface BankState {
  data: IBank[]
  totalData: number
  search: string
  limit: number
  offset: number
  id: number | null
  setData: (data: IBank[], totalData: number) => void
  setSearch: (search: string) => void
  setLimit: (limit: number) => void
  setOffset: (offset: number) => void
  setId: (id: number | null) => void
}

export const useBankStore = create<BankState>(set => ({
  data: [],
  totalData: 0,
  search: '',
  limit: 10,
  offset: 0,
  id: null,
  setData: (data, totalData) => set({ data, totalData }),
  setSearch: search => set({ search }),
  setLimit: limit => set({ limit }),
  setOffset: offset => set({ offset }),
  setId: id => set({ id })
}))
```

---

## Pola 2 — Extended (dengan sidebar & loading)

Tambahkan `sidebarOpen` dan `loading` jika halaman punya drawer sidebar:

```ts
sidebarOpen: boolean
setSidebarOpen: (open: boolean) => void
loading: boolean
setLoading: (loading: boolean) => void
```

---

## Pola 3 — Batch Setter

Update banyak field sekaligus → cegah multiple re-render:

```ts
setMerchantSelection: (selection: IMerchantSelection) => set({
  id: selection.id,
  companyId: selection.companyId,
  branchId: selection.branchId,
  merchantIds: selection.merchantIds,
  companyName: selection.companyName,
  branchName: selection.branchName
})
```

---

## Pola 4 — Socket Data Merge

```ts
setSocketData: (data: Partial<DashboardData>) =>
  set(state => ({
    data: state.data
      ? { ...state.data, ...data }
      : (data as DashboardData)
  }))
```

---

## Pola 5 — Store dengan Reset

```ts
reset: () => set({
  data: [],
  totalData: 0,
  search: '',
  limit: 10,
  offset: 0
})
```

---

## Cara Akses Store

### Di Komponen (React Hook)
```ts
const { data, totalData, search, setSearch, setOffset } = useXxxStore()

// Selective subscription
const data = useXxxStore(state => state.data)
```

### Di Luar Komponen — Service / Socket Handler
```ts
// Di service
const state = useXxxStore.getState()
useXxxStore.getState().setData(list, total)

// Di socket handler
socket.on('message', (data) => {
  useXxxStore.getState().setSocketData(data)
})
```

**Jangan** pakai hook `useXxxStore()` di socket handler — menyebabkan stale closure.

---

## Store Global

### Snackbar — `useSnackbarStore`
```ts
import { useSnackbarStore } from 'src/store/snackbar.state'

// Di komponen
const snackbar = useSnackbarStore()
snackbar.showMessage({ message: 'Berhasil', severity: 'success', key: Date.now() })

// Di luar komponen
useSnackbarStore.getState().showMessage({ message: 'Berhasil', severity: 'success', key: Date.now() })
```

### Dialog Konfirmasi — `useDialogStore`
```ts
import useDialogStore from 'src/store/dialog.state'

const dialog = useDialogStore()
dialog.openDialog('Hapus Data', 'Yakin ingin menghapus?')
dialog.closeDialog()
```

### Access Menu — `useAdminStore`
```ts
import { useAdminStore } from 'src/store/admin.state'

const { accessMenu } = useAdminStore()
const hasMenu = useAdminStore.getState().accessMenu.some(m => m.name === 'Client')
```

---

## Aturan Penting

| Situasi | Cara Akses |
|---------|-----------|
| Di dalam komponen React | `const { x } = useXxxStore()` |
| Di service / async function | `useXxxStore.getState().x` |
| Di socket `on('event')` handler | `useXxxStore.getState().setX(val)` |
| Update banyak field sekaligus | Buat batch setter |
| Reset saat logout | Panggil `reset()` dari `handleLogout` di `AuthContext` |

$ARGUMENTS
