---
name: datagrid
description: Referensi pola MUI DataGrid di project ini — definisi columns + CellType, RowOptions action menu, server-side pagination dengan Zustand store, sortModel, auto-numbering baris, dan struktur halaman list lengkap. Gunakan saat membuat halaman CRUD baru atau menambah kolom/aksi ke tabel yang ada.
---

# DataGrid — MUI DataGrid Reference

---

## Import

```ts
import { DataGrid, GridRenderCellParams } from '@mui/x-data-grid'
```

---

## Pola Lengkap — Halaman List dengan DataGrid

### 1. Interface CellType

Definisikan `CellType` di atas `columns` menggunakan tipe entity dari `src/types/`:

```ts
import { IMyDomain } from 'src/types/data/my-domain'

interface CellType {
  row: IMyDomain
}
```

---

### 2. Definisi Columns

```ts
const columns = [
  // Kolom nomor urut (auto dari offset + index)
  {
    sortable: false,
    field: 'lineNo',
    headerName: 'No',
    flex: 0,
    editable: false,
    renderCell: (params: GridRenderCellParams<number>) =>
      params.row.lineNo + params.api.getRowIndex(params.id) + 1
  },

  // Kolom teks biasa
  {
    flex: 0.2,
    minWidth: 200,
    field: 'name',
    headerName: 'Name',
    renderCell: ({ row }: CellType) => (
      <Typography noWrap variant='subtitle1'>{row.name}</Typography>
    )
  },

  // Kolom status (contoh: boolean / is_active)
  {
    flex: 0.1,
    minWidth: 100,
    field: 'is_active',
    headerName: 'Status',
    renderCell: ({ row }: CellType) => (
      <Typography variant='body2' color={row.is_active ? 'success.main' : 'error.main'}>
        {row.is_active ? 'Active' : 'Inactive'}
      </Typography>
    )
  },

  // Kolom actions (selalu di paling kanan, sortable: false)
  {
    flex: 0.1,
    minWidth: 90,
    sortable: false,
    field: 'actions',
    headerName: 'Actions',
    renderCell: ({ row }: CellType) => <RowOptions id={row.id} />
  }
]
```

---

### 3. RowOptions — Action Menu per Baris

```tsx
import { useState } from 'react'
import { IconButton, Menu, MenuItem } from '@mui/material'
import IconifyIcon from 'src/@core/components/icon'
import useDialogStore from 'src/store/dialog.state'
import { useMyDomainStore } from 'src/store/my-domain.state'
import { getMyDomainById } from 'src/services/my-domain.service'

const RowOptions = ({ id }: { id: number }) => {
  const [anchorEl, setAnchorEl] = useState<null | HTMLElement>(null)
  const [editData, setEditData] = useState<any>(null)
  const [dialogEdit, setDialogEdit] = useState(false)

  const store = useMyDomainStore(state => state)
  const dialog = useDialogStore(state => state)

  const rowOptionsOpen = Boolean(anchorEl)

  const handleDeleteClick = () => {
    dialog.openDialog('Delete Item', 'Are you sure you want to delete this item?')
    store.setId(id)
    setAnchorEl(null)
  }

  return (
    <>
      <IconButton size='small' onClick={e => setAnchorEl(e.currentTarget)}>
        <IconifyIcon icon='mdi:dots-vertical' />
      </IconButton>

      <Menu
        keepMounted
        anchorEl={anchorEl}
        open={rowOptionsOpen}
        onClose={() => setAnchorEl(null)}
        anchorOrigin={{ vertical: 'bottom', horizontal: 'right' }}
        transformOrigin={{ vertical: 'top', horizontal: 'right' }}
        PaperProps={{ style: { minWidth: '8rem' } }}
      >
        <MenuItem
          sx={{ '& svg': { mr: 2 } }}
          onClick={async () => {
            setDialogEdit(true)
            const res = await getMyDomainById(id)
            setEditData(res)
            setAnchorEl(null)
          }}
        >
          <IconifyIcon icon='mdi:edit-outline' />
          Edit
        </MenuItem>
        <MenuItem sx={{ '& svg': { mr: 2 } }} onClick={handleDeleteClick}>
          <IconifyIcon icon='mdi:delete-outline' />
          Delete
        </MenuItem>
      </Menu>

      {/* Dialog edit di-render di dalam RowOptions agar punya akses ke editData */}
      <DialogEditMyDomain isOpen={dialogEdit} useDialogEdit={setDialogEdit} data={editData} />
    </>
  )
}
```

---

### 4. Komponen Halaman Utama

```tsx
import { Box, Card, FormControl, Grid, TextField, Typography } from '@mui/material'
import { DataGrid } from '@mui/x-data-grid'
import { useEffect, useState } from 'react'
import CustomButton from 'src/components/custom/buttonCustom'
import { getMyDomainList } from 'src/services/my-domain.service'
import { useMyDomainStore } from 'src/store/my-domain.state'
import DialogAddMyDomain from 'src/components/dialog/my-domain/dialogAdd'
import ConfirmDialogMyDomain from 'src/components/dialog/my-domain/confirmDialog'

const MyDomainPage = () => {
  const [dialogAdd, setDialogAdd] = useState(false)
  const [sortModel, setSortModel] = useState([])

  const store = useMyDomainStore(state => state)

  // Search dengan debounce sederhana
  const handleSearch = (e: React.ChangeEvent<HTMLInputElement>) => {
    setTimeout(() => store.setSearch(e.target.value), 500)
  }

  // Pagination
  const handlePageSizeChange = (size: number) => {
    store.setLimit(size)
    store.setOffset(0)  // reset ke halaman pertama saat page size berubah
  }

  const handlePageChange = (page: number) => {
    store.setOffset(page * store.limit)
  }

  // Fetch data saat parameter berubah
  useEffect(() => {
    getMyDomainList()
  }, [store.search, store.limit, store.offset])

  // Tambahkan lineNo dari offset untuk nomor urut yang benar di tiap halaman
  const rows = store.data.map(item => ({ ...item, lineNo: store.offset }))

  return (
    <Grid sx={{ width: '100%' }}>
      <Card>
        <Box sx={{ p: 5 }}>
          <Typography variant='h5' sx={{ fontWeight: 'bold', color: 'primary.main' }}>
            My Domain
          </Typography>
        </Box>
        <Card>
          <Box sx={{ p: 5, maxWidth: '300px' }}>
            <Box sx={{ pb: '20px' }}>
              <FormControl fullWidth>
                <TextField
                  size='small'
                  placeholder='Search...'
                  variant='outlined'
                  onChange={handleSearch}
                />
              </FormControl>
            </Box>
            <CustomButton
              width='100%'
              name='Add My Domain'
              type='button'
              disabled={false}
              onClick={() => setDialogAdd(true)}
            />
            <DialogAddMyDomain isOpen={dialogAdd} useDialog={setDialogAdd} />
            <ConfirmDialogMyDomain />
          </Box>

          <DataGrid
            autoHeight
            rows={rows}
            columns={columns}
            pageSize={store.limit}
            rowCount={store.totalData}
            rowsPerPageOptions={[10, 25, 50]}
            onPageSizeChange={handlePageSizeChange}
            onPageChange={handlePageChange}
            page={Math.floor(store.offset / store.limit)}
            paginationMode='server'
            sortModel={sortModel}
            onSortModelChange={(model: any) => setSortModel(model)}
            sx={{ '& .MuiDataGrid-columnHeaders': { borderRadius: 0 } }}
          />
        </Card>
      </Card>
    </Grid>
  )
}

export default MyDomainPage
```

---

## Props DataGrid — Ringkasan

| Prop | Nilai | Keterangan |
|------|-------|-----------|
| `autoHeight` | — | Tinggi menyesuaikan jumlah baris |
| `rows` | `store.data` dengan `lineNo` | Selalu inject `lineNo: store.offset` |
| `columns` | array definisi kolom | Definisikan di luar komponen |
| `pageSize` | `store.limit` | Ukuran halaman dari store |
| `rowCount` | `store.totalData` | Total data untuk pagination server |
| `rowsPerPageOptions` | `[10, 25, 50]` | Pilihan page size |
| `paginationMode` | `'server'` | Selalu server-side |
| `page` | `Math.floor(store.offset / store.limit)` | Halaman aktif (0-based) |
| `onPageChange` | `page => store.setOffset(page * store.limit)` | Update offset |
| `onPageSizeChange` | `size => { store.setLimit(size); store.setOffset(0) }` | Reset ke halaman 1 |
| `sortModel` / `onSortModelChange` | `useState([])` | Sorting lokal |

---

## Kolom Khusus yang Sering Dipakai

### Kolom tanggal
```ts
{
  flex: 0.15,
  minWidth: 150,
  field: 'created_at',
  headerName: 'Created At',
  renderCell: ({ row }: CellType) => (
    <Typography variant='body2'>
      {row.created_at ? dayjs(row.created_at).format('DD MMM YYYY') : '-'}
    </Typography>
  )
}
```

### Kolom angka/currency
```ts
{
  flex: 0.15,
  minWidth: 130,
  field: 'amount',
  headerName: 'Amount',
  renderCell: ({ row }: CellType) => (
    <Typography variant='body2'>
      {Number(row.amount).toLocaleString('id-ID', { style: 'currency', currency: 'IDR' })}
    </Typography>
  )
}
```

### Kolom chip/badge
```ts
{
  flex: 0.1,
  minWidth: 100,
  field: 'status',
  headerName: 'Status',
  renderCell: ({ row }: CellType) => (
    <Chip
      label={row.status}
      color={row.status === 'active' ? 'success' : 'default'}
      size='small'
    />
  )
}
```

---

## Auto-numbering Baris

```ts
// Di definisi kolom
renderCell: (params: GridRenderCellParams<number>) =>
  params.row.lineNo + params.api.getRowIndex(params.id) + 1

// Di rows (inject lineNo = offset halaman ini)
const rows = store.data.map(item => ({ ...item, lineNo: store.offset }))
```

Hasilnya: halaman 1 (offset 0) → baris 1,2,3…; halaman 2 (offset 10) → baris 11,12,13…

---

## Aturan

| Situasi | Solusi |
|---------|--------|
| Pagination | Selalu `paginationMode='server'` — data list dari BE |
| `columns` letaknya | Di luar komponen (bukan di dalam function) — cegah re-create tiap render |
| Dialog edit | Render di dalam `RowOptions` agar punya akses ke `editData` lokal |
| Dialog confirm delete | Render sekali di halaman induk, state dari `useDialogStore` |
| Search | Debounce 500ms dengan `setTimeout`, panggil `store.setSearch()` |
| Reset pagination saat search | `setOffset(0)` saat `setSearch` atau `setLimit` |

$ARGUMENTS
