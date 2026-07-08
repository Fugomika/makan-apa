# CLAUDE.md — MakanApa

## Project Overview
Mobile web app that helps users pick what to eat through a step-by-step choice flow.
Single file, no build step, no backend, no login. All data lives in **localStorage** and is
export/importable as JSON.

- **To run:** `open MakanApa/index.html` (or serve over HTTP for browser testing)

---

## Stack
- Single file: `index.html`
- `<script type="module">` — plain vanilla JS, no external imports, no CDN dependencies
- Google Fonts: **Nunito** (weights 400, 600, 700, 800, 900)
- **localStorage** for all state (menu + tempat + harga). Export/Import JSON for backup / pindah device.
- No bundler, no build step, no auth, no network calls.

---

## Design System

Reference: modern mobile NFT marketplace UI — bright, kawaii, clean, colorful cards.

| Token           | Value                                      |
|-----------------|--------------------------------------------|
| Background      | `#C8FF00` (electric lime green)            |
| Card            | `#FFFFFF`, `border-radius: 28px`           |
| Font            | Nunito, 800–900 weight for headings        |
| Primary button  | Dark `#111111` pill on intro; lime on result |
| Text highlight  | Yellow `#FFE400` gradient underline (`.hl` class) |
| Choice cards    | White card, pastel gradient top image area |
| Doodle accents  | `✦ ✧ ♥ + ★` symbols, `opacity: 0.12`, CSS animated |
| Shadow          | `0 8px 32px rgba(0,0,0,0.10)`              |

### Pastel families (for `.cc-img`)
`.p-yellow` Nasi · `.p-violet` Non Nasi · `.p-sky` Berkuah · `.p-peach` Non Kuah ·
`.p-mint` Lanjut · `.p-lavender` Random · `.p-rose` Pedas · `.p-sage` Tidak Pedas ·
`.p-salt` Gurih · `.p-honey` Manis

---

## User Flow

```
Intro (top bar: ⚙️ Kelola Menu)
  └─ s1: Nasi / Non Nasi
       └─ s2: Berkuah / Non Kuah
            └─ s3: Mode?
                 ├─ Lanjut Pilih
                 │    └─ s4: Pedas / Tidak Pedas
                 │         └─ s5: Gurih / Manis → Result
                 └─ Random Aja → Result (gabungan 4 kombo rasa×taste)

Result: makanan + daftar TEMPAT (tree) dengan filter budget (💰) + 🎲 re-roll
Kelola Menu (s-manage): list makanan, tambah/edit/hapus, Export/Import JSON
Form Makanan (s-food-form): emoji, nama, deskripsi, kategori, editor tempat + harga
```

---

## Data Model & Storage

Semua menu disimpan di `localStorage['makanapa_v1']`:

```js
// shape:
{ version: 1, foods: {
    '<base>-<kuah>-<rasa>-<taste>': [ Food, ... ],   // 16 possible category keys
} }
// Food:  { e: emoji, n: name, d: desc, places: Place[] }
// Place: { name: string, price: 1|2|3|4 }
```

- **Seed:** objek `DEFAULT_FOODS` (16 kategori × 6 = 96 makanan, tanpa places) dipakai sekali saat
  localStorage kosong. Ini hanya benih — sumber kebenaran runtime adalah `store` di localStorage.
- **`store`** = variabel modul, di-load `loadStore()` saat boot.
- **Harga (kemahalan):** 4 level → `PRICE_LABEL = [null,'Hemat','Sedang','Mahal','Sultan']`,
  ikon `moneyIcons(p) = '💰'.repeat(p)`.

### Category key parsing
Key = `base-kuah-rasa-taste`, tapi `base` (`non-nasi`) dan `rasa` (`tidak-pedas`) mengandung `-`.
Jangan `split('-')` langsung. Pakai `splitCat(key)` yang memakai daftar nilai `DIMS` untuk parse
greedy dari depan (base) dan belakang (taste). `CAT_LABEL[key]` menyimpan label human-readable.

---

## JS Architecture

`<script type="module">` — fungsi yang dipakai `onclick` di HTML di-expose lewat `window.*`.

State:
```js
const st = { base, kuah, mode, rasa, taste };  // pilihan alur
let pool = [], idx = 0;                          // hasil acak + posisi re-roll
let budgetFilter = 0;                            // 0 = semua, else 1..4
let store = { version:1, foods:{} };             // sumber data (localStorage)
let editing = null;                              // { catKey, idx } saat edit form
```

| Fungsi | Tugas |
|---|---|
| `go(id)` | ganti screen; set `.app.top` untuk layar panjang (manage/form) |
| `pick(key,val,next)` | set `st[key]`, navigate atau `showResult()` |
| `buildPool()` | array makanan dari `store.foods` (gabung 4 kombo bila `mode==='random'`) |
| `showResult()` | build pool (guard kosong), reset budget, render, confetti |
| `render(f)` | isi emoji/nama/desc/stat 2×2, panggil `renderPlaces(f)` |
| `renderBudgetChips()` / `setBudget(lvl)` | chip filter budget di layar hasil |
| `renderPlaces(f)` | daftar tempat terfilter+terurut; hint bila kosong |
| `reroll()` / `restart()` | putar makanan lain / reset ke intro |
| `loadStore` / `saveStore` | baca/tulis localStorage (dengan `normalize`) |
| `normalize(obj)` | validasi & bersihkan struktur import (buang tempat tanpa nama, clamp harga 1–4) |
| `exportJSON()` / `importJSON(file)` | download / upload JSON menu |
| `showManage()` / `renderManageList()` | layar Kelola Menu + daftar makanan |
| `openFoodForm(catKey?,idx?)` / `openFoodFormFor(food)` | buka form tambah/edit |
| `splitCat(key)` | parse key kategori jadi `[base,kuah,rasa,taste]` |
| `addPlaceRow(place?)` / `collectPlaces()` | kelola baris tempat di form |
| `saveFoodForm()` / `deleteFoodFromForm()` | simpan (termasuk pindah kategori) / hapus |
| `confetti(n)` / `showToast(msg)` | efek + notifikasi |

---

## Food Database (seed)

`DEFAULT_FOODS['{base}-{kuah}-{rasa}-{taste}']` → array `{ e, n, d }` (places ditambah `[]` saat seed).
16 kategori (8 gurih + 8 manis) × 6 = **96 item**. Menambah di seed hanya memengaruhi user baru
(atau setelah clear localStorage); user existing edit lewat **Kelola Menu**.

---

## Adding / Editing (runtime)
Lewat layar **Kelola Menu** → **Tambah Makanan** / tap kartu untuk edit. Tiap makanan bisa diberi
banyak **tempat**, tiap tempat punya **harga 1–4**. Simpan → tersimpan ke localStorage.
**Backup/pindah device:** Export JSON, lalu Import di device lain.

---

## Security / notes
- Semua render nama makanan/tempat pakai `textContent` / DOM node (bukan `innerHTML` string) —
  data berasal dari import user, jadi hindari celah injeksi.
- `importJSON` selalu `try/catch` + `normalize`; JSON invalid/tak dikenali tidak pernah disimpan.
- Tidak ada secret/kunci di file — aman untuk repo publik & GitHub Pages.
