# Moonhop — Mini Platformer

Game platformer 2D sederhana menggunakan Vue 3, TypeScript, Vite, dan HTML Canvas.

## Fitur
- Gerakan kiri/kanan, gravitasi, dan lompat
- Platform, duri/rintangan, koin, checkpoint, portal tujuan
- 3 level dengan layout berbeda
- Pause, reset, skor, progres, best score di localStorage
- Kontrol keyboard dan tombol layar sentuh
- UI dark mode bergaya indie/pixel game

## Menjalankan project

Pastikan Node.js sudah terpasang. Buka terminal di folder project:

```bash
npm install
npm run dev
```

Vite akan menampilkan alamat lokal, biasanya `http://localhost:5173`.

Untuk build produksi:

```bash
npm run build
npm run preview
```

## Kontrol
- `A` / `←`: bergerak ke kiri
- `D` / `→`: bergerak ke kanan
- `W` / `↑` / `Space`: lompat
- `Esc`: pause/resume
- `R`: reset level

Catatan: permainan dirender pada Canvas; UI menu dan pemilihan level dibuat dengan Vue.
