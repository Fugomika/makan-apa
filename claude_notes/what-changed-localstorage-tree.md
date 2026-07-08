# MakanApa: dari Supabase ke localStorage + tree tempat + filter budget

Catatan plain-language soal perombakan besar `index.html`. Tujuannya biar kamu paham
*apa* yang berubah, *kenapa* begitu, dan *bagaimana* bagian-bagiannya nyambung.

---

## Kenapa dirombak

Versi lama butuh **login Supabase** untuk menyimpan favorit, dan seluruh menu di-hardcode di JS
(tidak bisa kamu ubah tanpa edit kode). Kamu mau:

1. **Tanpa login.** Data kamu sendiri, disimpan lokal, bisa di-**backup lewat JSON**.
2. **Struktur tree.** Satu makanan (mis. "Ayam Goreng") bisa punya banyak **tempat** favorit.
3. **Menyesuaikan dompet.** Tiap tempat punya **tingkat kemahalan**, jadi hasil bisa difilter budget.

---

## Apa yang berubah

### 1. Supabase dibuang total
Hilang: `import createClient`, konstanta `SUPABASE_*`, semua fungsi auth (`submitAuth`, `logout`,
`updateAuthUI`, …) dan favorites (`saveFavorite`, `showFavoritesScreen`), plus layar `#s-auth` &
`#s-favorites`. Efek samping bagus: **tidak ada lagi kunci rahasia di file** — aman untuk repo publik.

### 2. Menu pindah ke localStorage
- Kunci: `localStorage['makanapa_v1']`.
- Bentuk data:
  ```js
  { version:1, foods: { '<base>-<kuah>-<rasa>-<taste>': [ {e,n,d,places:[{name,price}]} ] } }
  ```
- Objek `DB` lama **diganti nama jadi `DEFAULT_FOODS`** dan cuma dipakai sebagai *benih* saat
  localStorage masih kosong (`loadStore()` → kalau kosong/invalid, `seedStore()`).
  Jadi 96 makanan lama tidak perlu ditulis ulang — mereka jadi isi awal.
- Kenapa `store` jadi satu variabel modul (bukan konstanta): karena sekarang **data bisa berubah
  saat runtime** (user tambah/edit), lalu `saveStore()` menuliskannya kembali ke localStorage.

### 3. Export / Import JSON (backup)
- `exportJSON()` — bikin `Blob` dari `store`, trigger download `makanapa-menu.json` lewat anchor +
  `URL.createObjectURL`. Ini cara standar "download file dari JS" tanpa server.
- `importJSON(file)` — `FileReader` baca file → `JSON.parse` **di dalam try/catch** →
  `normalize()` memvalidasi & membersihkan (buang tempat tanpa nama, clamp harga ke 1–4, kasih
  default emoji). **Kalau gagal parse atau bentuknya asing, data lama TIDAK ditimpa** — cuma toast error.
  Ini penting: import dari file luar itu untrusted, jadi harus divalidasi sebelum disimpan.

### 4. Struktur tree: makanan → tempat → harga
- Tiap `Food` sekarang punya `places: [{ name, price }]`. `price` 1–4 dipetakan ke label
  `PRICE_LABEL = [null,'Hemat','Sedang','Mahal','Sultan']` dan ikon `'💰'.repeat(price)`.
- Di layar hasil, `renderPlaces(food)` menampilkan tempat-tempatnya (terurut naik harga).
  Kalau makanan belum punya tempat, muncul hint + tombol "＋ Tambah tempat" yang lompat ke editor.

### 5. Filter budget di layar hasil
- State `budgetFilter` (0 = semua, else 1..4). `renderBudgetChips()` menggambar chip
  `[Semua] [💰] [💰💰] [💰💰💰] [💰💰💰💰]`; `setBudget(lvl)` mengubah filter lalu render ulang tempat.
- Filter murni **di layar hasil** (bukan step tambahan di alur) — kamu bisa ganti budget bolak-balik
  tanpa mengulang pertanyaan.

### 6. Editor "Kelola Menu"
- `#s-manage`: daftar makanan (bisa difilter per kategori), tombol Export/Import, Tambah Makanan.
- `#s-food-form`: form emoji + nama + deskripsi + **4 dropdown kategori** + editor baris tempat
  (nama + select harga + hapus). `saveFoodForm()` menangani 3 kasus: tambah baru, edit di kategori
  sama, dan **pindah kategori** (hapus dari kategori lama, push ke kategori baru).

---

## Satu jebakan yang perlu diingat: parsing kunci kategori

Kunci kategori = `base-kuah-rasa-taste`, TAPI `base` bisa `non-nasi` dan `rasa` bisa `tidak-pedas` —
keduanya mengandung tanda `-`. Jadi **`key.split('-')` salah** (akan pecah jadi 6 bagian).

Solusinya `splitCat(key)`: ambil `base` dengan mencocokkan awalan dari daftar nilai yang valid
(`nasi` / `non-nasi`), `taste` dari akhiran (`gurih` / `manis`), sisanya (`kuah-rasa`) dipecah lagi
dengan cara yang sama. Label human-readable tiap kunci sudah dihitung di depan ke `CAT_LABEL`.

---

## Cara nyoba cepat
1. Serve foldernya (`python3 -m http.server`) lalu buka di browser.
2. Kelola Menu → Tambah Makanan "Ayam Goreng" + 3 tempat (harga 1/2/3) → Simpan.
3. Jalankan alur sampai dapat Ayam Goreng → cek daftar tempat + toggle budget.
4. Export JSON → hapus localStorage (DevTools) → reload (menu default kembali) → Import file tadi.
