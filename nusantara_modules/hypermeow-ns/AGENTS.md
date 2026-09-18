# hypermeow-ns — panduan buat AI agent

Panduan lengkap modul protokol WhatsApp+Signal ini ada di **[README.md](README.md)** (arsitektur, cara pasang, `kirim_bootstrap`/`proses_node`/dll, media, reaksi & polling, app state, dan bagian **"Catatan implementasi"** yang isinya jebakan protokol + concurrency yang udah pernah bikin bot ini gagal total kalau dilanggar).

Kalau lagi kerja di bot yang PAKAI modul ini (bukan modulnya sendiri) — misalnya `ndlabs-bot` — panduan level plugin/fitur ada di [`AGENTS.md` root repo](../../AGENTS.md), bukan di sini.

Ringkasan super singkat concurrency (detail lengkap di README.md § Catatan implementasi):

1. Cuma SATU thread yang boleh manggil `ws_terima` (loop koneksi utama).
2. `tunggu_iq` gak boleh dipanggil sinkron dari loop itu sendiri — wajib lewat `jalan(...)`.
3. Semua `jalankan_perintah(...)` (curl/ffmpeg/dll) wajib dibatasi waktu (`--max-time` atau bungkus `timeout -s KILL N`).
4. Kalau nemu freeze total (semua thread `futex_do_wait` di gdb, restart count gak nambah), pastiin binary `nusa` yang dipake udah ngandung fix AB-BA `chan->mu`/`gcMutex_` (README.md § Catatan implementasi).
