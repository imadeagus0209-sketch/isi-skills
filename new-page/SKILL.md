---
name: new-page
description: Scaffold halaman list read-only dengan filter bar + DataGrid. Cocok dipakai setelah /new-service. Argumen: <domain> <parent-path>
---

# New Page Scaffold (Read-only List)

Buatkan halaman list untuk domain: **$ARGUMENTS**

Format argumen: `<domain> <parent-path>`
Contoh: `credit-balance /transaction` atau `tag-history /master`

---

Buat dua file berikut:

## 1. View — `src/views/pages/<parent-path>/<domain>/index.tsx`

### Import
```tsx
import { Icon } from '@iconify/react'
import { Box, Button, Card, Grid, TextField, Typography } from '@mui/material'
import { DataGrid } from '@mui/x-data-grid'
import dayjs from 'dayjs'
import { useEffect, useState } from 'react'
import { get<Domain>List } from 'src/services/<parent-path>/<domain>.service'
import { use<Domain>Store } from 'src/store/<parent-path>/<domain>.state'
import { I<Domain> } from 'src/types/data/<parent-path>/<domain>'
```

### Struktur komponen
```tsx
const <Domain>View = () => {
  const { data, totalData, loading, limit, offset, search, setSearch, setLimit, setOffset } = use<Domain>Store()

  const [localSearch, setLocalSearch] = useState(search)

  const handleApplyFilter = () => { setOffset(0) }
  const handleResetFilter = () => { /* reset ke default */ }

  useEffect(() => {
    const timer = setTimeout(() => setSearch(localSearch), 500)
    return () => clearTimeout(timer)
  }, [localSearch, setSearch])

  useEffect(() => {
    get<Domain>List()
  }, [search, limit, offset])

  const rows = data.map((item, index) => ({ ...item, lineNo: offset + index + 1 }))

  return (
    <Grid container spacing={3}>
      <Grid item xs={12}>
        <Card>
          <Box sx={{ p: 4, display: 'flex', flexWrap: 'wrap', gap: 3, alignItems: 'flex-end' }}>
            <TextField size='small' label='Search' value={localSearch} onChange={e => setLocalSearch(e.target.value)}
              placeholder='...' sx={{ minWidth: 200 }}
              InputProps={{ startAdornment: <Icon icon='mdi:magnify' style={{ marginRight: 4, color: '#9ca3af' }} /> }}
            />
            <Button variant='contained' onClick={handleApplyFilter} startIcon={<Icon icon='mdi:filter' />}>Apply</Button>
            <Button variant='outlined' onClick={handleResetFilter} startIcon={<Icon icon='mdi:refresh' />}>Reset</Button>
          </Box>
        </Card>
      </Grid>
      <Grid item xs={12}>
        <Card>
          <DataGrid
            autoHeight
            rows={rows}
            columns={columns as any}
            loading={loading}
            disableSelectionOnClick
            paginationMode='server'
            rowCount={totalData}
            pageSize={limit}
            page={Math.floor(offset / limit)}
            rowsPerPageOptions={[10, 25, 50]}
            onPageChange={page => setOffset(page * limit)}
            onPageSizeChange={size => { setLimit(size); setOffset(0) }}
            rowHeight={64}
            sx={{ '& .MuiDataGrid-columnHeaders': { borderRadius: 0 } }}
          />
        </Card>
      </Grid>
    </Grid>
  )
}
```

### Kolom DataGrid
- Kolom `lineNo` untuk nomor urut
- Format angka IDR: `new Intl.NumberFormat('id-ID', { minimumFractionDigits: 2 }).format(val)`
- Format tanggal: `dayjs(val).format('DD MMM YYYY')` + baris kedua `dayjs(val).format('HH:mm')`
- Warna status negatif/positif: `color: isNegative ? '#dc2626' : '#16a34a'`

### Tipografi
Gunakan variant MUI standar — jangan override `fontFamily` secara manual.

| Kegunaan | Variant / sx |
|----------|-------------|
| Judul section/card | `variant='h5'` atau `variant='h6'` |
| Label kolom | `variant='body2' fontWeight={600}` |
| Nilai sekunder | `variant='body2' color='text.secondary'` |
| Caption/timestamp | `variant='caption' color='text.secondary'` |
| Kode / ID teknis | `variant='body2' sx={{ fontFamily: 'monospace', fontSize: '0.75rem' }}` |

---

## 2. Page — `src/pages/<parent-path>/<domain>/index.tsx`

```tsx
import <Domain>View from 'src/views/pages/<parent-path>/<domain>'

export default <Domain>View
```

---

## Checklist
- [ ] Store sudah ada (`use<Domain>Store`) — kalau belum, jalankan `/new-service` dulu
- [ ] Service sudah ada (`get<Domain>List`) — kalau belum, jalankan `/new-service` dulu
- [ ] Semua query param API sudah tercakup di filter bar

$ARGUMENTS
