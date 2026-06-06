---
name: validation
description: Referensi lengkap pola validasi form — Yup schema, React Hook Form + Controller, defaultValues, error display, dan custom validator yang dipakai di project ini. Gunakan saat membuat schema baru, menambah field validasi, atau debug error form.
---

# Validation — Yup + React Hook Form Reference

---

## Direktori Validasi

```
src/validation/
├── settings.validation.ts       # divisi, posisi, jadwal, company, branch
├── employee.validation.ts       # personal, kontak, pekerjaan, privilege
├── merchant.validasi.ts         # create/update merchant
├── admins.validation.ts         # admin user
├── device.validation.ts         # device EDC
├── device-edc.validation.ts     # EDC config
├── gate.validation.ts           # gate access
├── gateway.validation.ts        # LoRaWAN gateway
├── enddevice.validation.ts      # LoRaWAN end device
├── vehicle.validation.ts        # kendaraan
├── uniform.validation.ts        # seragam
├── item.validation.ts           # item/produk
├── camp.validasi.ts             # camp
├── customField.validation.ts    # field kustom
├── manageRoles.ts               # role & menu
├── submitTransaction.ts         # transaksi auth
└── paymentBlockchain.validation.ts
```

Naming: `<domain>.validation.ts` (beberapa lama pakai `.validasi.ts`)

---

## Setup Form — Pola Standar

```ts
import { useForm, Controller } from 'react-hook-form'
import { yupResolver } from '@hookform/resolvers/yup'
import * as yup from 'yup'

const schema = yup.object().shape({
  name: yup.string().required('Name is required').max(100, 'Max 100 characters'),
  email: yup.string().email('Invalid email format').required('Email is required')
})

const { control, handleSubmit, reset, formState: { errors } } = useForm({
  defaultValues: { name: '', email: '' },
  mode: 'onChange',
  resolver: yupResolver(schema)
})

const onSubmit = (data: typeof defaultValues) => {
  // panggil service di sini
}
```

---

## Pola Controller (MUI TextField)

```tsx
<Controller
  name="merchantName"
  control={control}
  render={({ field }) => (
    <TextField
      {...field}
      fullWidth
      label="Merchant Name"
      error={!!errors.merchantName}
      helperText={errors.merchantName?.message}
    />
  )}
/>
```

### Controller untuk Select (MUI)
```tsx
<Controller
  name="gender"
  control={control}
  render={({ field }) => (
    <FormControl fullWidth error={!!errors.gender}>
      <InputLabel>Gender</InputLabel>
      <Select {...field} label="Gender">
        <MenuItem value="Male">Male</MenuItem>
        <MenuItem value="Female">Female</MenuItem>
      </Select>
      <FormHelperText>{errors.gender?.message}</FormHelperText>
    </FormControl>
  )}
/>
```

---

## Validator Umum yang Dipakai di Project

### String wajib, tidak boleh hanya spasi
```ts
yup.string()
  .required('Field is required')
  .test('not-only-spaces', 'Cannot contain only spaces', value =>
    value ? value.trim().length > 0 : false
  )
```

### Nomor telepon (10–15 digit)
```ts
yup.string()
  .required('Phone number is required')
  .max(15, 'Phone Number can be at most 15 characters')
  .test('is-valid-number', 'Submit a valid Phone number', value => {
    if (!value) return false
    return /^[0-9]{10,15}$/.test(value)
  })
```

### NIK / Identity Number (10–16 digit)
```ts
yup.number()
  .typeError('Identity number must be a number')
  .required('Identity number is required')
  .test('is-valid-number', 'Submit a valid Identity number', value => {
    if (!value) return false
    return /^[0-9]{10,16}$/.test(value.toString())
  })
```

### Latitude & Longitude
```ts
lat: yup.number()
  .typeError('Latitude must be a number')
  .required('Latitude is required')
  .min(-90, 'Latitude minimal -90')
  .max(90, 'Latitude maksimal 90'),

long: yup.number()
  .typeError('Longitude must be a number')
  .required('Longitude is required')
  .min(-180, 'Longitude minimal -180')
  .max(180, 'Longitude maksimal 180')
```

### Number nullable (bisa kosong, tidak wajib)
```ts
yup.number()
  .nullable()
  .transform((v, orig) => (orig === '' || orig === null || orig === undefined ? null : v))
  .min(0, 'Minimal 0')
  .max(100, 'Maksimal 100')
```

### Date range (start tidak boleh setelah end)
```ts
workPeriodFrom: yup.date()
  .required('From Work Period is required')
  .test('workPeriodFrom', 'Start date cannot be after end date', function (value) {
    const workPeriodTo = this.parent.workPeriodTo
    return !value || !workPeriodTo || !dayjs(value).isAfter(dayjs(workPeriodTo))
  }),

workPeriodTo: yup.date()
  .required('Date is required')
  .test('workPeriodTo', 'End date cannot be earlier than start date', function (value) {
    const workPeriodFrom = this.parent.workPeriodFrom
    return !value || !workPeriodFrom || !dayjs(value).isBefore(dayjs(workPeriodFrom))
  })
```

### Array of objects (nested)
```ts
adminId: yup.array().of(
  yup.object().shape({
    adminName: yup.string().nullable()
  })
)
```

### Object nested dengan custom test
```ts
areaAccess: yup.object().shape({
  gate: yup.array(),
  camp: yup.array(),
  canteen: yup.array()
})
.test('is-valid-number', 'Please select at least one option!', value => {
  if (!value.gate || !value.camp || !value.canteen) return false
  return value.gate.length > 0 || value.camp.length > 0 || value.canteen.length > 0
})
.required('Please select at least one option!')
```

### Field wajib hanya jika field lain bernilai tertentu (conditional)
```ts
commission: yup.string()
  .nullable()
  .test('commission-required', 'Commission is required', function (val) {
    const branch = this.parent?.branch
    if (branch) return true  // skip jika branch = true
    return val !== null && val !== undefined && val !== ''
  })
```

---

## Reset Form

```ts
// Reset ke defaultValues
reset()

// Reset ke nilai tertentu (saat edit)
reset({
  name: data.name,
  email: data.email
})

// Reset saat dialog tutup
useEffect(() => {
  if (!isOpen) reset()
}, [isOpen])
```

---

## Pola Multi-Step Form (beberapa schema terpisah)

Gunakan schema berbeda per step — lihat `employee.validation.ts`:

```ts
// Step 1
export const addEmployeePersonalSchema = yup.object().shape({ ... })
// Step 2
export const addContactSchema = yup.object().shape({ ... })
// Step 3
export const addEmployeeWorkSchema = yup.object().shape({ ... })
```

Masing-masing step punya `useForm` sendiri dengan resolver schema yang sesuai.

---

## defaultValues — Selalu Definisikan

Eksport `defaultValue` bersamaan dengan schema agar konsisten antara create/reset:

```ts
export const defaultValueMerchant = {
  merchantName: '',
  email: '',
  phone: '',
  lat: 0,
  long: 0,
  fee: null,
  commission: null
}
```

---

## Aturan

| Situasi | Solusi |
|---------|--------|
| Field angka bisa kosong | Tambahkan `.nullable().transform(...)` |
| Validasi lintas field | Gunakan `this.parent` di dalam `.test()` |
| Field tidak wajib tapi ada format | Chain `.optional()` atau skip `.required()` |
| Date dari dayjs | Gunakan `yup.date().typeError('Invalid date format')` |
| Multi-step form | Schema terpisah per step, bukan satu schema besar |

$ARGUMENTS
