---
name: check
description: Jalankan TypeScript check, ESLint, dan git diff sebelum commit. Laporan siap atau tidak untuk commit.
---

# Pre-commit Check

Jalankan pengecekan kode sebelum commit untuk project ini.

## Langkah-langkah:

### 1. TypeScript Check
```bash
npx tsc --noEmit --skipLibCheck
```
Tampilkan semua error TypeScript yang ditemukan. Jika ada error, jelaskan penyebab dan cara fixnya.

### 2. ESLint Check
```bash
yarn lint
```
Tampilkan semua warning/error ESLint.

### 3. Ringkasan Git Diff
```bash
git diff --stat HEAD
```
Tampilkan file apa saja yang berubah.

### 4. Laporan
Berikan ringkasan:
- ✅ Jika semua bersih — siap commit
- ❌ Jika ada error — tampilkan daftar masalah yang perlu diperbaiki sebelum commit

$ARGUMENTS
