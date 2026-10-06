# Tutorial: Lumos custom tool (Figma → tokens → site)

Template ini mengubah Lumos for Astro dari skala fluid (`clamp()`) menjadi nilai tetap per breakpoint yang diambil dari Figma, supaya hasilnya bisa dibuat 1:1.

## 1. Konsep singkat

| Breakpoint | Lebar | Cara kerja |
| --- | --- | --- |
| mobile | ≤ 767px | default |
| tablet | 768–991px | `@media (width >= 48rem)` |
| desktop | ≥ 992px | `@media (width >= 62rem)` |

- Semua token responsif punya tiga angka (px tanpa satuan) di `src/styles/base.css`:
  ```css
  --h1: calc((var(--bp-mobile) * var(--h1-mobile) + var(--bp-tablet) * var(--h1-tablet) + var(--bp-desktop) * var(--h1-desktop)) / 16 * 1rem);
  --h1-mobile: 32;
  --h1-tablet: 45;
  --h1-desktop: 54;
  ```
  Mengubah desain berarti mengubah angka, bukan rumus.
- Line height juga tiga angka per breakpoint (`--h1-line-height-mobile/-tablet/-desktop`), bukan rasio.
- Spacing, radius, dan icon memakai nama Figma: `--space-1-5rem`, `--radius-0-5rem`, `--icon-m`, dan seterusnya. Skala spacing dan padding Figma disatukan.
- Warna dua lapis: primitif `--color-<grup>-<step>` (palet Figma) lalu semantik di blok tema (`--background`, `--text`, `--brand`).
- Trim teks mati secara default. Nyalakan dengan class `.text-trim` pada elemen atau induknya.
- Alat ukurnya: skill `lumos-import-figma` (`.agents/skills/lumos-import-figma/`). Script `convert.mjs` membaca token langsung dari `base.css`, jadi tidak ada salinan yang bisa usang.

## 2. Memulai project dari template

```bash
gh repo create nama-project --template myuzara02/astro-lumos-custom --clone
cd nama-project
npm install
npx astro dev --background      # server di background
npx astro dev status            # lihat port
npx astro dev stop              # matikan
```

Atau klik **Use this template** di halaman repo GitHub.

Nilai bawaan template (palet safron-mango, margin 112, dan seterusnya) milik desain asalnya. Langkah berikutnya menggantinya dengan desain Anda.

## 3. Menyambungkan Figma ke omp

1. Buka Figma desktop, buka file desain, masuk **Dev Mode**.
2. Aktifkan **MCP server** di panel kanan Dev Mode (alamat `http://127.0.0.1:3845/mcp`).
3. Daftarkan di omp. Tambahkan ke `~/.omp/agent/mcp.json` (berlaku di semua project) atau `.omp/mcp.json` (khusus project):
   ```json
   {
     "mcpServers": {
       "figma-desktop": { "type": "http", "url": "http://127.0.0.1:3845/mcp" }
     }
   }
   ```
4. Di omp jalankan `/mcp reload`, lalu `/mcp test figma-desktop`. Kalau tool belum muncul, buka sesi baru.

Catatan: server remote Figma (`mcp.figma.com`) memakai OAuth dan menolak pendaftaran klien omp, jadi pakai server desktop. Figma desktop harus tetap terbuka selama slicing.

## 4. Slicing, urutan kerja

Jalankan dari root project. Semua perintah memakai `node .agents/skills/lumos-import-figma/convert.mjs`, disingkat `convert` di bawah.

### 4.1 Variabel Figma → token

1. Export koleksi variabel Figma sebagai JSON (mis. `Responsive.json` dengan mode desktop/tablet/mobile, dan `Static.json` untuk warna dan font). Simpan di root project. File ini sengaja di-gitignore.
2. Jalankan:
   ```bash
   convert --variables Responsive.json Static.json
   ```
3. Hasilnya tabel per variabel: `match`, `DIFFERS` (kedua nilai ditampilkan), atau `MISSING`. Bagian `TO UPDATE BY HAND` dan `TO PLACE BY HAND` berisi baris CSS yang harus Anda taruh di `base.css`.
4. Mode dikenali dari namanya (tidak peka huruf besar, salah ketik `dekstop` ditoleransi). Grup variabel yang tidak dikenal dilaporkan, tidak diabaikan.
5. Bagian `FONTS` membandingkan font Figma dengan `fonts:` di `astro.config.mjs` (lihat bagian 6).

Skill tidak menulis `base.css`. Anda atau agen menempatkan token sesuai tabel penempatan di `SKILL.md`.

### 4.2 Layout: margin, gutter, padding section

Variabel Figma tidak memuat ini, jadi ukur dari geometri node.

1. Minta agen memanggil `get_metadata` untuk node halaman (section Figma yang memuat frame Desktop, Tablet, Mobile), lalu simpan XML-nya, misalnya `page.xml`.
2. Jalankan:
   ```bash
   convert --metadata page.xml
   convert --metadata page-a.xml page-b.xml        # beberapa halaman sekaligus
   convert --metadata page.xml --wrapper 123:456   # bila wrapper tidak terdeteksi
   ```
3. Hasil: `site-margin`, padding section atas dan bawah, `site-gutter` per breakpoint, plus tingkat keyakinan dan outlier. Lalu perbandingan dengan `base.css`: `MATCH`, `DIFFERS`, atau `UNMAPPED`.
4. Breakpoint dikenali dari lebar frame, bukan nama. Wadah konten dicari dari struktur (anak yang lebih sempit dan terinset), jadi nama `global-wrapper` tidak wajib.
5. Skill tidak memutuskan padding itu `small`, `medium`, atau `large`. Anda yang memetakan. `Section` memakai `medium` sebagai default.
6. Hero dan cta sering tidak punya anak di metadata. Bacalah via `get_design_context`. Hati-hati: angka fallback di sana (`var(--padding/4_5rem,72px)`) adalah nilai mode desktop. Geometri metadata yang jadi acuan.

Contoh format uji tersedia di `.agents/skills/lumos-import-figma/fixtures/sample-page.xml`:
```bash
convert --metadata .agents/skills/lumos-import-figma/fixtures/sample-page.xml
```

### 4.3 Tipografi dari text node

Letter spacing dan text-transform tidak ada di variabel, hanya di text node. Kumpulkan per style, lalu masukkan ke JSON:

```json
{
  "type": [{
    "name": "H1",
    "sizePx": { "desktop": 54, "tablet": 45, "mobile": 32 },
    "lineHeightPx": { "desktop": 60, "tablet": 48, "mobile": 36 },
    "letterPx": { "desktop": -1.35, "tablet": -1.125, "mobile": -0.8 }
  }],
  "layout": [
    { "token": "section-space-medium", "px": { "desktop": 80, "tablet": 64, "mobile": 56 } }
  ]
}
```
```bash
convert --json design.json
```

Letter spacing disimpan sebagai satu nilai `em` per style. Kalau antar breakpoint berbeda, skill bertanya dan tidak menerapkannya.

### 4.4 Pencarian cepat

```bash
convert --px 30                 # snap ke token spacing terdekat (desktop)
convert --px 30 --bp mobile
convert --lh 36/32              # line height 36 pada font 32
convert --color "#FFFFFF@60"    # warna beralpha → color-mix
```

## 5. Aturan penting selama slicing

- Skill melaporkan selisih dan bertanya. Ia tidak membuat token baru diam-diam untuk menutupi ketidakkonsistenan desain.
- Selisih seperti weight H1 Bold di Figma versus Medium di template adalah keputusan per project. Template menyimpan default, dan hasil slicing menimpanya.
- Nilai yang tidak diukur (mis. tablet saat hanya desktop yang ada) ditandai sebagai tebakan di laporan.

## 6. Font

Figma biasanya memakai "Inter Display". Itu Inter pada optical size 32, bukan family terpisah di Google. Template sudah memuatnya:

```js
// astro.config.mjs
fonts: [{
  name: "Inter",
  cssVariable: "--font-inter",
  provider: fontProviders.google(),
  weights: ["400 700"],
  styles: ["normal"],
  options: { experimental: { variableAxis: { opsz: ["32"] } } },
}],
```

- Font diunduh dari Google saat dev atau build, jadi butuh internet.
- Untuk font lain, ganti `name`, `weights`, dan `variableAxis`. Untuk font berlisensi, pakai `fontProviders.local()` dengan `variants` per weight.
- `--variables` memberi tahu bila family atau weight Figma belum dikonfigurasi.

## 7. Memverifikasi hasil

1. Jalankan `npx astro dev --background` dan buka halaman.
2. Ubah lebar jendela di batas 767/768 dan 991/992px. Nilai harus melompat di titik itu.
3. Cek tipe: `npx astro check`.
4. Bandingkan dengan screenshot Figma di tiga lebar (1440, 834, 393). Mismatch biasanya berarti desain tidak konsisten (pertanyaan), bukan bug token.

## 8. Membangun halaman

Ikuti `LUMOS.md`: susun dari komponen (`Section`, `ContentWrapper`, `Heading`, `Paragraph`), jangan membuat class baru kecuali tidak ada varian yang cocok. Varian teks yang tersedia: `display`, `h1`–`h6`, `large`, `main`, `small`, `xsmall`, `overline-small`, `overline-main`. Ikon: `small`, `medium`, `large`, `2xs`–`4xl`.

## 9. Memperbarui skill di project lama

Project dari template adalah salinan. Perubahan template tidak ikut otomatis. Untuk membawa skill terbaru, salin folder `.agents/skills/lumos-import-figma` dari template. Versinya ada di `SKILL_VERSION` pada `convert.mjs`. Untuk framework Lumos sendiri gunakan skill `lumos-upgrade-version`.

## 10. Masalah umum

| Gejala | Penyebab dan solusi |
| --- | --- |
| `/mcp reauth` gagal "OAuth authorization failed" | Server remote Figma menolak klien omp. Pakai server desktop (bagian 3). |
| Tool Figma tidak muncul | Jalankan `/mcp reload`, atau buka sesi baru. Pastikan Figma desktop terbuka dan MCP server aktif. |
| Heading terlihat Regular, bukan Bold | Weight belum dimuat. Cek `weights` di `astro.config.mjs`. |
| Padding section tablet tidak cocok dengan variabel | Desainer memasang variabel berbeda per frame. Ukur dari metadata. |
| `--metadata` melaporkan "not measurable" | Section berupa instance atau tanpa anak. Baca via `get_design_context`. |
| Teks tombol lebih tinggi dari desain | Trim mati. Memang disengaja, mengikuti line box Figma. Nyalakan dengan `.text-trim` bila perlu. |
