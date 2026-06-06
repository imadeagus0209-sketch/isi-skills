---
name: types
description: Referensi konvensi TypeScript types di project ini — pola IResponseXxx, ICreateXxx, IUpdateXxx, IXxx entity, struktur IResponse<T>, direktori src/types/data/, dan cara organisasi per domain. Gunakan saat membuat type baru untuk domain/fitur baru.
---

# TypeScript Types — Conventions Reference

---

## Direktori Types

```
src/types/data/
├── response.ts              # IResponse<T> — wrapper generic semua response API
├── auth.ts                  # IRequestLogin, IResponseLogin
├── employee.ts              # ICreateEmployee, IEditEmployee, dll.
├── position.ts              # IPositionSettings, IResponsePosition, ICratePosition
├── transaction.ts           # IResponseTransaction, dll.
├── transaction-edc.ts
├── master/                  # Types untuk halaman master
│   ├── bank.ts              # IBankMaster, IResponseBankMaster, ICreateBankMaster
│   ├── merchant.ts          # IMerchantMaster, IResponseMerchantMaster
│   ├── device.ts            # IDeviceMaster, IResponseDeviceMaster, IDeviceCraete
│   └── ...
├── settings/                # Types untuk halaman settings
├── transaction/             # Types untuk dashboard & transaksi
├── lorawan/
└── plantation/
```

Naming file: `<domain>.ts` (flat) atau `<folder>/<domain>.ts` (nested jika banyak domain)

---

## IResponse — Wrapper Generic API

Semua response API dibungkus dengan `IResponse<T>` dari `src/types/data/response.ts`:

```ts
// src/types/data/response.ts
export interface IResponse<T> {
  headers: Headers
  data: T
}

export interface Headers {
  statusCode: number
  message: string
}
```

Cara pakai di service:
```ts
import { AxiosResponse } from 'axios'
import { IResponse } from 'src/types/data/response'

const response: AxiosResponse<IResponse<IResponseMyDomain>> = await http.get(api.myDomain.getList)
// Akses data: response.data.data
// Akses message: response.data.headers.message
```

---

## Pola Type per Domain

Setiap domain punya 3–4 interface standar:

### Pola sederhana (contoh: position)
```ts
// src/types/data/position.ts

// 1. Response list dari API
export interface IResponsePosition {
  listPositionMaster: IPositionSettings[]
  totalData: number
}

// 2. Entity (shape data satu item)
export interface IPositionSettings {
  id: number
  position_name: string
  create_at: string
  update_at: string
}

// 3. Payload create
export interface ICreatePosition {
  position_name: string
}

// 4. Payload update (extend create + wajib id)
export interface IUpdatePosition extends ICreatePosition {
  id: number
}
```

### Pola domain lebih kompleks (contoh: device)
```ts
// src/types/data/master/device.ts

export interface IResponseDeviceMaster {
  listEdc: IDeviceMaster[]
  totalData: number
}

export interface IDeviceMaster {
  id: number
  is_active: number
  mac_address: string
  name: string
  online: boolean
  unique_code?: string       // optional — tidak selalu ada di response
  convert_amount?: number
}

export interface ICreateDevice {
  mac_address: string
  name: string
  unique_code: number
}
```

### Pola dengan nested object
```ts
export interface ICutoff {
  start_cutoff_time: string
  end_cutoff_time: string
  withdrawal_time: string
}

export interface ICreateBankMaster {
  id?: number
  bankName: string
  cutoff: ICutoff[]         // array of nested object
}

export interface IBankMaster {
  id: number
  bank_name: string
  bank_code: string
  branch_name?: string
  account_number?: string
  is_active: boolean
  created_at?: string
  updated_at?: string
  cutoff?: ICutoff[]
}
```

---

## Konvensi Naming

| Interface | Pola | Contoh |
|-----------|------|--------|
| Response list | `IResponse<Domain>` | `IResponsePosition`, `IResponseDeviceMaster` |
| Entity item | `I<Domain>` | `IPositionSettings`, `IDeviceMaster`, `IBankMaster` |
| Payload create | `ICreate<Domain>` | `ICreatePosition`, `ICreateDevice` |
| Payload update | `IUpdate<Domain>` | `IUpdatePosition` (biasanya extend ICreate + id) |
| Form values | `I<Domain>Form` | `IPositionForm` (opsional, jika beda dari payload) |

**Catatan:** Beberapa file lama menggunakan typo (`ICratePosition` bukan `ICreatePosition`) — ikuti naming yang benar untuk file baru.

---

## Nama Field — Konsistensi BE vs FE

BE sering menggunakan `snake_case` untuk nama field (`position_name`, `is_active`, `created_at`).  
FE menggunakan nama apa adanya dari response BE — **jangan rename** field di type, karena akan dipakai langsung dari `response.data.data`.

```ts
// Benar: ikuti nama field dari BE
export interface IPositionSettings {
  id: number
  position_name: string   // snake_case dari BE
  create_at: string
}

// Di komponen — akses langsung
<Typography>{row.position_name}</Typography>
```

Payload ke BE (create/update) boleh camelCase jika BE menerimanya:
```ts
export interface ICreatePosition {
  positionName: string   // camelCase jika BE minta camelCase
}
```

---

## IResponse<T> dalam AxiosResponse

```ts
import { AxiosResponse } from 'axios'
import { IResponse } from 'src/types/data/response'
import { IResponsePosition, IPositionSettings } from 'src/types/data/position'

// GET list
const response: AxiosResponse<IResponse<IResponsePosition>> = await http.get(api.position.getList)
const list: IPositionSettings[] = response.data.data.listPositionMaster
const total: number = response.data.data.totalData

// GET item by id
const response: AxiosResponse<IResponse<IPositionSettings>> = await http.get(api.position.getItem, { params: { id } })
const item: IPositionSettings = response.data.data

// POST / PUT — ambil message dari headers
const response = await http.post(api.position.create, payload)
const message: string = response.data.headers.message
```

---

## Type untuk Store Zustand

Store selalu punya shape standar menggunakan type entity:

```ts
// src/store/my-domain.state.ts
import { IMyDomain } from 'src/types/data/my-domain'
import { create } from 'zustand'

interface MyDomainState {
  data: IMyDomain[]          // list dari API
  totalData: number
  id: number | null          // id yang dipilih (untuk delete)
  search: string
  limit: number
  offset: number
  setData: (data: IMyDomain[], totalData: number) => void
  setId: (id: number | null) => void
  setSearch: (search: string) => void
  setLimit: (limit: number) => void
  setOffset: (offset: number) => void
}
```

---

## Type untuk Form (React Hook Form)

Form values mengacu ke payload create, bukan entity:

```ts
// Definisikan inline di komponen dialog, atau export dari validation file
type FormValues = {
  name: string
  email: string
  phone?: string   // opsional di form
}

const { control, handleSubmit } = useForm<FormValues>({
  defaultValues: { name: '', email: '', phone: '' }
})
```

---

## Aturan

| Situasi | Solusi |
|---------|--------|
| Type list response | Buat `IResponseXxx` dengan `listXxx: IXxx[]` dan `totalData: number` |
| Optional field dari BE | Tandai dengan `?` — misal `unique_code?: string` |
| Extend untuk update | `IUpdateXxx extends ICreateXxx { id: number }` |
| Nested object berulang | Pisahkan jadi interface sendiri (contoh: `ICutoff`) |
| Field snake_case dari BE | Pertahankan nama aslinya di type, jangan rename |
| Type form ≠ type entity | Buat type form terpisah jika shape berbeda dari entity |
| Lokasi file | `src/types/data/<domain>.ts` atau `src/types/data/<folder>/<domain>.ts` |

$ARGUMENTS
