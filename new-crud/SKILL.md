---
name: new-crud
description: Scaffold lengkap fitur CRUD baru (type, validation, service, store, sidebar, table, page). Argumen: <domain> <endpoint-prefix>
---

# New CRUD Feature Scaffold

Buatkan scaffold lengkap untuk fitur CRUD baru di project ini berdasarkan argumen yang diberikan: **$ARGUMENTS**

Format argumen: `<domain> <endpoint-prefix>`
Contoh: `bank /bank` atau `device-edc /edc`

---

Ikuti konvensi project ini dan buat semua file berikut:

## 1. API Endpoint — `src/configs/api.ts`
Tambahkan key baru di object `api` (di bagian `master` jika relevan):
```ts
<domain>: {
  get: '<endpoint-prefix>/get-list',
  getById: '<endpoint-prefix>/get-item',
  create: '<endpoint-prefix>/create',
  update: '<endpoint-prefix>/update',
  delete: '<endpoint-prefix>/delete'
}
```

## 2. Type — `src/types/data/master/<domain>.ts`
Buat interface `I<Domain>` dengan field `id: number` dan field umum lainnya.

## 3. Validation — `src/validation/<domain>.validation.ts`
Buat schema Yup `create<Domain>Schema` menggunakan `yup.object().shape({})`.

## 4. Service — `src/services/master/<domain>.service.ts`
Buat 5 fungsi: `get<Domain>`, `get<Domain>ById`, `create<Domain>`, `update<Domain>`, `delete<Domain>`.
- Gunakan `http` dari `src/configs/http`
- Gunakan endpoint dari `src/configs/api`
- `get<Domain>` harus baca state dari store dan update store setelah fetch

## 5. Store — `src/store/master/<domain>.state.ts`
Buat Zustand store dengan:
- State: `data`, `totalData`, `sidebarOpen`, `id`, `search`, `limit`, `offset`
- Actions: `setData`, `setSearch`, `setLimit`, `setOffset`, `setSidebarOpen`, `setId`

## 6. View — Sidebar `src/views/pages/master/<domain>/sidebar.tsx`
Buat komponen `Sidebar<Domain>` menggunakan:
- `Drawer` anchor right, width 400
- React Hook Form + yupResolver
- `useSnackbarStore` untuk notifikasi
- Loading state pada tombol Submit
- Reset form saat drawer tutup

## 7. View — Table Header `src/views/pages/master/<domain>/tableHeader.tsx`
Buat komponen `TableHeader` dengan pola berikut:
- Jika halaman ini adalah sub-halaman (punya parent), tambahkan `onBack` prop → tampilkan header dengan `IconButton` arrow-left + `Typography` judul, lalu `Divider`, lalu baris search + tombol Add
- Jika halaman top-level, cukup baris search + tombol Add

Pattern dengan back button:
```tsx
interface TableHeaderProps {
  value: string
  toggle: () => void
  handleFilter: (val: string) => void
  onBack: () => void
}

const TableHeader = ({ value, toggle, handleFilter, onBack }: TableHeaderProps) => (
  <>
    <Box sx={{ p: { xs: 2, sm: 3, md: 5 }, display: 'flex', alignItems: 'center', gap: 1 }}>
      <IconButton onClick={onBack} sx={{ p: 0.5, color: 'primary.main', '&:hover': { backgroundColor: 'action.hover' } }}>
        <IconifyIcon icon='mdi:arrow-left' fontSize={28} />
      </IconButton>
      <Typography variant='h6' sx={{ fontWeight: 'bold', color: 'primary.main' }}>
        <Judul Halaman>
      </Typography>
    </Box>
    <Divider />
    <Box sx={{ p: 5, pb: 3, display: 'flex', flexWrap: 'wrap', alignItems: 'center', justifyContent: 'end' }}>
      <TextField size='small' value={value} sx={{ mr: 6, mb: 2 }} placeholder='Search...' onChange={e => handleFilter(e.target.value)} />
      <Button sx={{ mb: 2 }} onClick={toggle} variant='contained'>Add</Button>
    </Box>
  </>
)
```

Di page (`index.tsx`), pass `onBack` ke TableHeader:
```tsx
<TableHeader ... onBack={() => router.push('/parent-path')} />
```

## 8. Page — `src/pages/master/<domain>/index.tsx`
Buat halaman dengan:
- `DataGrid` dari MUI X
- Kolom: No, field utama, Actions (edit/delete)
- `RowOptions` component dengan menu edit & delete + ConfirmDialog
- Pagination server-side

## 9. Navigation
Tanyakan apakah perlu ditambahkan ke navigation menu di `src/navigation/`.

---

Pastikan semua file mengikuti pola yang sudah ada di codebase.

$ARGUMENTS
