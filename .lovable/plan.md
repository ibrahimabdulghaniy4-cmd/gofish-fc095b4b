# Pilih Karakter Setelah Connect Wallet

## Yang akan dirasakan pemain

1. Pemain connect wallet dan menandatangani pesan seperti sekarang (tidak ada yang berubah di langkah ini).
2. Tepat setelah profil siap, muncul kartu "Choose your angler" berisi 4 karakter. Setiap kartu menampilkan pratinjau 3D kecil yang bisa dilihat langsung, nama, dan satu baris deskripsi gaya.
3. Pemain memilih satu, tekan "Play". Karakter itu langsung dipakai di dunia (memancing, jalan, berenang, semua sama seperti sebelumnya).
4. Pilihan tersimpan ke wallet, jadi saat kembali karakternya sama. Di panel profil ada tombol "Change character" untuk menggantinya kapan saja.

## Empat karakter

Semuanya bergaya kotak (blocky) seperti karakter yang sudah ada, tapi beda total dari kepala ke sepatu:

| Karakter | Wajah & kulit | Rambut | Aksesori | Baju | Celana | Sepatu |
| --- | --- | --- | --- | --- | --- | --- |
| Rook (yang sekarang, jadi bawaan) | kulit hangat, mata garis | jabrik perak | headphone | kemeja gelap + vest & dasi | celana abu robek | sneaker putih |
| Marisol | kulit sawo, senyum + tahi lalat | kuncir kuda hitam panjang | topi jerami pita merah + kacamata di kepala | tanktop karang & apron nelayan | pendek denim | sandal karet kuning |
| Bjorn | kulit pucat, pipi merah, brewok | jambang & jenggot pirang tebal | beanie rajut + kacamata safety | sweater wol tebal bergaris | overall karet hijau | boot karet hitam tinggi |
| Kenji | kulit terang, mata tegas | undercut hitam berponi | ikat kepala + masker leher + tas selam kecil | jaket teknis biru bergaris neon | legging trek ramping | sneaker teknis abu-neon |

Perbedaan tidak hanya warna: bentuk rambut, bentuk penutup kepala, siluet badan (kurus/kekar), potongan lengan, dan bentuk kaki/sepatu berbeda per karakter.

## Menjaga sistem yang sudah jalan

- Semua gerak, animasi mancing, joran, ikan yang diangkat, kamera, dan suara tetap memakai jalur kode yang sama — hanya tampilan tubuh yang dipilih dari preset.
- Kalau data pilihan tidak ada, karakter bawaan (Rook) dipakai, jadi pemain lama tidak melihat perubahan apa pun.
- Kartu pilihan hanya muncul saat pemain belum pernah memilih; tidak menghalangi quest, toko, chat, leaderboard, atau dokumentasi.

## Detail teknis

- **Data**: migrasi baru menambah `profiles.character_id text not null default 'rook'` (nilai divalidasi ke daftar preset di server). Kolom baru bersifat opsional secara logika sehingga baris lama otomatis `rook`. Server function baru `setCharacter` di `src/lib/profile.functions.ts` — verifikasi tanda tangan wallet lewat `verifyProof` yang sudah ada, lalu update kolom via `supabaseAdmin`, dan mengembalikan baris profil untuk disimpan ke `useProfileStore`. Types Supabase di-regenerasi.
- **Preset**: `src/lib/characterLooks.ts` mengekspor `CHARACTER_PRESETS` (id, nama, deskripsi, palet: skin/hair/shirt/pants/shoe/accent, plus flag bentuk: `hairStyle`, `headwear`, `build`, `legwear`, `footwear`) dan `characterLook(id)` dengan fallback ke `rook` — pola sama dengan `rodLooks.ts`/`baitLooks.ts` yang sudah dipakai.
- **Render**: markup tubuh di `Angler.tsx` dipindahkan ke `src/components/game/AnglerBody.tsx` sebagai komponen yang menerima preset + semua ref (`torso`, `head`, `legL`, `legR`, `leftArm`, `rightArm`, `handAnchor`, `backAnchor`, `heldFishAnchor`) via props, sehingga seluruh logika `useFrame` di `Angler.tsx` tidak berubah sama sekali. Sub-bagian per preset dipecah jadi helper kecil (`Hair`, `Headwear`, `Torso`, `Legs`) di file yang sama.
- **UI pilih karakter**: `src/components/game/CharacterSelect.tsx` — dialog (komponen `ui/dialog` yang ada) dengan 4 kartu; tiap kartu punya `<Canvas>` kecil (kamera statis, `ambientLight` + `directionalLight`, tanpa post-processing) yang merender `AnglerBody` preset itu dalam pose berdiri, plus rotasi lambat. Sesuai anggaran performa: mesh kotak, tanpa shadow map di pratinjau.
- **Alur**: state `characterSelectOpen` di `useProfileStore`; dibuka otomatis oleh `StartGate` saat `profile` tersedia dan `profile.character_id` belum pernah di-set (kolom penanda `character_chosen_at timestamptz`), ditutup setelah pilihan tersimpan. `ProfilePanel` mendapat tombol "Change character" yang membuka dialog yang sama.
- **Verifikasi**: `tsgo --noEmit`, lalu Playwright headless — buka halaman, pastikan tidak ada error konsol, buka dialog pilih karakter secara langsung dan ambil screenshot 4 pratinjau.
