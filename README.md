# ndlabs-bot

Self-bot WhatsApp ditulis di Nusantara (`.ns`), jalan di atas modul protokol [UwUchan](https://github.com/NusaLang/UwUchan).

## Pasang

```
nusa get github.com/NusaLang/UwUchan
cp .env.example .env
```

Isi `.env` (nomor bot, nomor owner), lalu jalankan:

```
nusa main.ns
```

Panduan lengkap (arsitektur, cara nambah plugin, wa-api, concurrency/deadlock rules, gotcha) ada di **[AGENTS.md](AGENTS.md)**.
