# Menghilangkan sisa lag: pilihan kualitas grafis + laut & cahaya lebih ringan

## Yang akan didapat pemain

1. **Tombol pengaturan grafis** di pojok layar (ikon roda gigi, dekat tombol dompet) dengan tiga pilihan: **Rendah / Sedang / Tinggi**. Pilihan tersimpan di perangkat, jadi tidak perlu diatur ulang setiap main.
   - **Rendah**: tanpa efek kilau (bloom), tanpa bayangan, resolusi 1x, laut disederhanakan, cahaya bawah air lebih kecil.
   - **Sedang** (default di perangkat lemah/HP): bayangan aktif, bloom aktif, resolusi maks 1,25x.
   - **Tinggi**: seperti sekarang.
2. **Air lebih ringan di semua mode** — riak laut dihitung dua kali per titik gambar, bukan empat kali, dengan hasil visual yang praktis sama.
3. **Cahaya saat memancing tidak lagi menutupi layar** — ukuran cahaya bawah air, kilatan permukaan, dan pilar cahaya dikecilkan sehingga jauh lebih murah digambar, terutama di perangkat dengan grafis terintegrasi.

## Detail teknis

### 1. Store kualitas grafis
Buat `src/hooks/useGraphics.ts`: zustand store `{ tier: "low" | "medium" | "high", set(tier) }` dengan persistensi `localStorage` (`gofish.gfx`). Default: `medium` bila `navigator.hardwareConcurrency <= 4` atau layar sentuh, selain itu `high`. Ekspor turunan preset:

```ts
{ dpr: [1,1] | [1,1.25] | [1,1.5],
  shadows: false | true | true,
  bloom: false | true | true,
  multisampling: 0 | 2 | 2,
  oceanDetail: 0 | 1 | 2,   // dipakai shader laut
  glowScale: 0.55 | 0.8 | 1 }
```

### 2. GameCanvas
- `shadows={preset.shadows}`, `dpr={preset.dpr}`.
- `EffectComposer` hanya dirender bila `preset.bloom`; kalau tidak, tidak ada composer sama sekali (hindari pass resolve kosong). Karena composer bisa hilang, `gl={{ antialias: !preset.bloom }}` agar tanpa composer tetap ada AA murah — kecuali tier `low` yang tetap `antialias: false`.
- Ganti `key` pada `<Canvas>` mengikuti `tier` supaya perubahan shadows/antialias (opsi WebGL yang tidak reaktif) diterapkan lewat remount, satu kali saat pemain mengganti pilihan.

### 3. Ocean.tsx
- Hitung normal detail sekali: hilangkan pemanggilan `ripples` untuk `rz` dengan memakai turunan analitik dari dua sampel + `dFdx/dFdy` pada `r0` (`vec2 g = vec2(dFdx(r0), dFdy(r0)) / fwidth(p)` disederhanakan menjadi gradient dari satu sampel), sehingga total `ripples` per piksel turun dari 4 → 2 (`r0` untuk normal + `caustic`).
- Tambah uniform `uDetail` (dari `oceanDetail`): pada nilai 0 lewati oktaf ketiga `fbm` di `ripples` dan lewati blok caustic sepenuhnya (`ripples` per piksel jadi 1); pada 1 pakai 2 oktaf; pada 2 seperti sekarang (3 oktaf).
- Uniform diupdate lewat `useEffect` saat tier berubah, bukan per frame.

### 4. UnderwaterFishGlow.tsx
- Baca `glowScale` sekali per komponen dan kalikan pada semua `scale` sprite: `coreS`, `haloS`, `flashS`, `ring`, `beamWidth`/`beamLen`, serta bagian `animateCatchAscend` (`cs`, `hs`, `trailWidth/trailLen`).
- Turunkan angka dasar halo/flash/pilar ~25% untuk semua tier (halo 2.8 → 2.1, flash 2.6 → 2.0, beamWidth 1.4 → 1.1) agar overdraw aditif tidak lagi menutupi layar.
- Tidak mengubah logika fase memancing; hanya ukuran/opasitas visual.

### 5. Verifikasi
Jalankan harness Playwright: screenshot mode Tinggi dan Rendah, cek console bersih, dan pastikan laut + cahaya masih terlihat benar di kedua mode.

## Di luar cakupan
Tidak mengubah logika gameplay, ekonomi, data, atau `.env`.
