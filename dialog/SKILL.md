---
name: dialog
description: Referensi pola CRUD dialog di project ini — Add dialog (RHF + Yup), Edit dialog (setValue saat open), Confirm delete dialog via useDialogStore global, dan struktur direktori dialog. Gunakan saat membuat modal form baru, dialog hapus, atau dialog create/edit dalam satu komponen.
---

# Dialog — CRUD Modal Reference

---

## Direktori Dialog

```
src/components/dialog/
├── dialogConfirm.tsx          # Generic confirm dialog (props-based)
├── submitTransaction.tsx      # Auth dialog (username + password)
├── master/                    # Dialog untuk halaman master
├── settings/                  # Dialog untuk halaman settings
│   └── position/
│       ├── dialogAddPosition.tsx
│       ├── dialogEditPosition.tsx
│       └── confirmDialog.tsx
├── employee/
├── plantation/
└── transaction-edc/
```

Naming: `dialog<Action><Domain>.tsx` — contoh: `dialogAddPosition.tsx`, `dialogEditMerchant.tsx`

---

## Pola 1 — Add Dialog (Create)

```tsx
import { Fragment } from 'react'
import { Controller, useForm } from 'react-hook-form'
import { yupResolver } from '@hookform/resolvers/yup'
import { Button, Dialog, DialogActions, DialogTitle, Box, FormControl, TextField } from '@mui/material'
import { mySchema } from 'src/validation/my-domain.validation'
import { createMyDomain, getMyDomainList } from 'src/services/my-domain.service'
import { useSnackbarStore } from 'src/store/snackbar.state'

interface Props {
  isOpen: boolean
  useDialog: (value: boolean) => void
}

const DialogAddMyDomain = ({ isOpen, useDialog }: Props) => {
  const snackbar = useSnackbarStore()

  const { control, handleSubmit, reset, formState: { errors } } = useForm({
    defaultValues: { name: '' },
    resolver: yupResolver(mySchema),
    mode: 'onChange'
  })

  const onSubmit = async (data: any) => {
    try {
      const res = await createMyDomain(data)
      reset()
      useDialog(false)
      snackbar.showMessage({ message: res.headers.message, key: Date.now(), severity: 'success' })
      await getMyDomainList()
    } catch (error: any) {
      snackbar.showMessage({
        message: error.data?.headers?.message || 'An error occurred',
        key: Date.now(),
        severity: 'error'
      })
    }
  }

  return (
    <Fragment>
      <Dialog
        open={isOpen}
        onClose={() => { useDialog(false); reset() }}
        sx={{ '& .MuiDialog-paper': { width: '25%', maxWidth: 'unset' } }}
      >
        <DialogTitle sx={{ textAlign: 'center', fontWeight: 'bold' }}>Add My Domain</DialogTitle>
        <Box sx={{ display: 'flex', alignItems: 'center', justifyContent: 'center', p: 3 }}>
          <FormControl fullWidth sx={{ mb: 2 }}>
            <Controller
              name='name'
              control={control}
              render={({ field }) => (
                <TextField
                  {...field}
                  label='Name'
                  error={!!errors.name}
                  helperText={errors.name?.message}
                />
              )}
            />
          </FormControl>
        </Box>
        <DialogActions className='dialog-actions-dense'>
          <Button variant='contained' onClick={handleSubmit(onSubmit)}>Save</Button>
          <Button variant='contained' color='error' onClick={() => { useDialog(false); reset() }}>Cancel</Button>
        </DialogActions>
      </Dialog>
    </Fragment>
  )
}

export default DialogAddMyDomain
```

---

## Pola 2 — Edit Dialog

Perbedaan dari Add: terima `id`/`data` lewat props, gunakan `setValue` di `useEffect` saat dialog terbuka.

```tsx
interface Props {
  isOpen: boolean
  useDialogEdit: (value: boolean) => void
  data: IMyDomain | null   // data row yang di-edit
}

const DialogEditMyDomain = ({ isOpen, useDialogEdit, data }: Props) => {
  const snackbar = useSnackbarStore()

  const { control, handleSubmit, reset, setValue, formState: { errors } } = useForm({
    defaultValues: { name: '' },
    resolver: yupResolver(mySchema),
    mode: 'onChange'
  })

  // Populate form saat dialog terbuka dengan data yang akan di-edit
  useEffect(() => {
    if (!data) return
    setValue('name', data.name)
  }, [data])

  const onSubmit = async (formData: any) => {
    try {
      const res = await updateMyDomain({ ...formData, id: data?.id })
      reset()
      useDialogEdit(false)
      snackbar.showMessage({ message: res.headers.message, key: Date.now(), severity: 'success' })
      await getMyDomainList()
    } catch (error: any) {
      snackbar.showMessage({
        message: error.data?.headers?.message || 'An error occurred',
        key: Date.now(),
        severity: 'error'
      })
    }
  }

  return (
    <Fragment>
      <Dialog
        open={isOpen}
        onClose={() => useDialogEdit(false)}
        sx={{ '& .MuiDialog-paper': { width: '25%', maxWidth: 'unset' } }}
      >
        <DialogTitle sx={{ textAlign: 'center', fontWeight: 'bold' }}>Edit My Domain</DialogTitle>
        <Box sx={{ display: 'flex', alignItems: 'center', justifyContent: 'center', p: 3 }}>
          <FormControl fullWidth sx={{ mb: 2 }}>
            <Controller
              name='name'
              control={control}
              render={({ field }) => (
                <TextField
                  {...field}
                  label='Name'
                  error={!!errors.name}
                  helperText={errors.name?.message}
                />
              )}
            />
          </FormControl>
        </Box>
        <DialogActions className='dialog-actions-dense'>
          <Button variant='contained' onClick={handleSubmit(onSubmit)}>Save</Button>
          <Button variant='contained' color='error' onClick={() => useDialogEdit(false)}>Cancel</Button>
        </DialogActions>
      </Dialog>
    </Fragment>
  )
}
```

---

## Pola 3 — Confirm Delete Dialog (via useDialogStore global)

Dialog hapus tidak perlu props `isOpen` — state-nya dari `useDialogStore` global.

### Step 1: Di RowOptions — trigger dialog
```ts
import useDialogStore from 'src/store/dialog.state'
import { useMyDomainStore } from 'src/store/my-domain.state'

const dialog = useDialogStore(state => state)
const store = useMyDomainStore(state => state)

const handleDeleteClick = () => {
  dialog.openDialog('Delete My Domain', 'Are you sure you want to delete this item?')
  store.setId(id)   // simpan id ke store untuk dipakai saat confirm
  setAnchorEl(null)
}
```

### Step 2: Buat ConfirmDialog component
```tsx
import useDialogStore from 'src/store/dialog.state'
import { useMyDomainStore } from 'src/store/my-domain.state'
import { deleteMyDomain, getMyDomainList } from 'src/services/my-domain.service'
import { useSnackbarStore } from 'src/store/snackbar.state'

const ConfirmDialogMyDomain = () => {
  const state = useDialogStore(state => state)
  const store = useMyDomainStore(state => state)
  const snackbar = useSnackbarStore(state => state)

  const handleConfirm = async () => {
    try {
      const res = await deleteMyDomain(store.id as number)
      snackbar.showMessage({ message: res.headers.message, key: Date.now(), severity: 'success' })
      await getMyDomainList()
      state.closeDialog()
    } catch (error: any) {
      snackbar.showMessage({
        message: error.data?.headers?.message || 'An error occurred',
        key: Date.now(),
        severity: 'error'
      })
      state.closeDialog()
    }
  }

  return (
    <Dialog open={state.isOpen} onClose={state.closeDialog}>
      <DialogTitle>{state.title}</DialogTitle>
      <DialogContent>
        <DialogContentText>{state.message}</DialogContentText>
      </DialogContent>
      <DialogActions className='dialog-actions-dense'>
        <Button variant='contained' onClick={handleConfirm}>Confirm</Button>
        <Button variant='contained' color='error' onClick={state.closeDialog}>Cancel</Button>
      </DialogActions>
    </Dialog>
  )
}

export default ConfirmDialogMyDomain
```

### Step 3: Render di halaman list (sekali, di luar DataGrid)
```tsx
<ConfirmDialogMyDomain />
```

---

## Pola 4 — Dialog Lebih Besar (maxWidth sm/md)

Untuk form dengan banyak field — gunakan `maxWidth` dan `fullWidth`:

```tsx
<Dialog open={isOpen} onClose={handleClose} maxWidth='sm' fullWidth>
  <DialogTitle>
    <Box display='flex' justifyContent='space-between' alignItems='center'>
      <Typography variant='h6'>{isEditMode ? 'Edit' : 'Add'} Item</Typography>
      <IconButton onClick={handleClose}>
        <Icon icon='mdi:close' />
      </IconButton>
    </Box>
  </DialogTitle>
  <DialogContent>
    <form onSubmit={handleSubmit(onSubmit)}>
      <Box sx={{ display: 'flex', flexDirection: 'column', gap: 3, pt: 2 }}>
        {/* fields di sini */}
        <Box sx={{ display: 'flex', justifyContent: 'flex-end', gap: 2, mt: 3 }}>
          <Button variant='outlined' color='secondary' onClick={handleClose}>Cancel</Button>
          <Button variant='contained' type='submit'>
            {isEditMode ? 'Update' : 'Save'}
          </Button>
        </Box>
      </Box>
    </form>
  </DialogContent>
</Dialog>
```

---

## Pola 5 — Create & Edit dalam Satu Komponen

```tsx
interface Props {
  isOpen: boolean
  onClose: () => void
  editData?: IMyDomain | null  // undefined/null = create mode
}

const DialogMyDomain = ({ isOpen, onClose, editData }: Props) => {
  const isEditMode = !!editData

  useEffect(() => {
    if (isOpen && editData) {
      setValue('name', editData.name)
    } else if (isOpen && !editData) {
      reset()
    }
  }, [isOpen, editData])

  const onSubmit = async (data: any) => {
    if (isEditMode) {
      await updateMyDomain({ ...data, id: editData!.id })
    } else {
      await createMyDomain(data)
    }
    // ...
  }

  return (
    <Dialog open={isOpen}>
      <DialogTitle>{isEditMode ? 'Edit' : 'Add'} My Domain</DialogTitle>
      {/* ... */}
    </Dialog>
  )
}
```

---

## useDialogStore — API

```ts
import useDialogStore from 'src/store/dialog.state'

// Trigger dialog (dari RowOptions)
dialog.openDialog('Dialog Title', 'Dialog message/question here')

// Tutup dialog (dari confirm handler atau cancel)
dialog.closeDialog()

// Baca state
const { isOpen, title, message } = useDialogStore(state => state)
```

---

## Aturan

| Situasi | Solusi |
|---------|--------|
| Dialog delete | Pakai `useDialogStore` global — tidak perlu state lokal |
| Dialog create/edit | State lokal (`useState<boolean>`) di halaman induk |
| Populate form saat edit | `setValue` di dalam `useEffect` dengan deps `[data]` |
| Reset form saat tutup | Panggil `reset()` di handler `onClose` |
| Refresh list setelah submit | `await getMyDomainList()` setelah service berhasil |
| Error dari API | `error.data?.headers?.message` (bukan `error.message`) |

$ARGUMENTS
