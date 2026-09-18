# hypermeow-ns

Implementasi protokol WhatsApp Web dari nol pakai Nusantara (`.ns`). Gak pakai
`whatsmeow` atau library WhatsApp lain pas runtime — semua wire format,
handshake, dan kripto-nya ditulis sendiri.

Yang dipakai dari luar cuma 4 binary Go kecil di `tools/` buat hal yang belum
ada primitif-nya di Nusantara (signing Ed25519, zlib inflate, WebSocket
long-lived). Sisanya murni `.ns`.

## Pasang

Taruh foldernya di `nusantara_modules/hypermeow-ns/` dalam project kamu:

```
projectmu/
├── main.ns
└── nusantara_modules/
    └── hypermeow-ns/
```

`tools/wsrelay` harus jalan duluan sebelum bot dinyalain:

```bash
./nusantara_modules/hypermeow-ns/tools/wsrelay -addr 127.0.0.1:9520
```

Butuh plugin `crypto`, `sqlite`, sama `ws` dari Nusantara.

## Pakai

`client.ns` nyimpen semua alur runtime. `main.ns` kamu tinggal siapin konteks
terus panggil 4 fungsi ini:

```
buat KL = impor("nusantara_modules/hypermeow-ns/client.ns");

KL.kirim_bootstrap(k);                          // encrypt-count, passive->active, presence
jalan(fungsi() { KL.mulai_keepalive(k); });     // keepalive w:p tiap ~22 detik
KL.proses_node(k, node);                        // dispatcher tiap node masuk
KL.kirim_balasan_grup(k, group_user, teks, 0);  // kirim pesan ke grup
```

`k` itu satu peta yang isinya state koneksi. Isi minimal:

| Kunci | Isi |
|---|---|
| `ws_id` | id sesi dari `ws_buka` |
| `noise_socket` | hasil `handshake.lakukan_handshake` |
| `parser` | hasil `socket/framesocket.parser_baru()` |
| `session_data` | hasil `store/session.muat_sesi("main")` |
| `identity_kp` | keypair identity |
| `noise_kp` | keypair noise |
| `our_registration_id` | registration id device |
| `our_signed_prekey_priv` | private signed-prekey |
| `retry_ctx` | hasil `msgsend.siapkan_konteks_retry(...)` |
| `waBot` | objek bot kamu, harus punya `HandleMessage(sender, teks, from_me)` |
| `DB` | modul database kamu, harus punya `SimpanLIDJID(lid, jid)` |
| `pesan_grup_terkirim` | `peta_baru()`, cache pesan grup buat retry |
| `epoch` | peta bareng, `epoch["now"]` naik tiap reconnect |
| `epoch_saya` | nilai `epoch["now"]` pas koneksi ini dibuka |

`epoch` dipakai biar goroutine keepalive dari koneksi lama mati sendiri pas
reconnect — tanpa ini thread-nya numpuk terus.

Contoh lengkap ada di `main.ns` repo ndlabs-bot.

## Isi folder

```
client.ns        alur runtime: bootstrap, keepalive, dispatcher node, kirim pesan grup
msgsend.ns       Signal protocol: X3DH, Double Ratchet, Sender Key, bikin/parse node
handshake.ns     Noise_XX handshake
pair_code.ns     pairing pakai kode 8 digit
pair_success.ns  verifikasi + balesan pair-success
group.ns         bikin/keluar grup, kelola peserta, subject/deskripsi, link undangan
user.ns          usync, cek nomor, info user, foto profil, blokir, privasi
presence.ns      presence, langganan presence, chat state (typing)
media.ns         enkripsi/dekripsi media, media_conn, unduh & unggah
msgsecret.ns     reaksi & polling (message secret + AES-GCM)
notification.ns  urai notifikasi grup, device, foto profil, prekey
newsletter.ns    channel: info, ikuti, bisukan, reaksi (lewat w:mex)
call.ns          urai panggilan masuk, tolak panggilan
broadcast.ns     privasi status, daftar broadcast
appstate.ns      sync app state: ambil patch, dekode mutasi, LTHash

binary/          binary XML WhatsApp (encode, decode, token table)
socket/          framing + noise transport
store/           SQLite: sesi, sender key, sesi kirim, prekey, device grup
util/            AES-GCM, AES-CBC, HKDF, HMAC-SHA512, base32, keypair
proto/           protobuf: client payload, ADV, handshake
types/           JID
appstate/        app state keys, LTHash
tools/           binary Go pembantu (lihat bawah)
```

## tools/

Empat binary Go, dipanggil lewat `jalankan_perintah` atau lewat socket:

- `keygen_identity` — bikin identity keypair + signed prekey + signature
- `sign_identity` — tanda tangan XEdDSA (`ecc.CalculateSignature`)
- `zlib_inflate` — dekompresi frame `w:g2` yang di-zlib
- `wsrelay` — relay WebSocket lokal (`127.0.0.1:9520`) ke `wss://web.whatsapp.com/ws/chat`,
  pakai `coder/websocket` (library yang sama kayak whatsmeow)

## Catatan implementasi

Empat hal ini beda tipis dari kelihatannya tapi bikin protokolnya gagal total
kalau salah. Semuanya udah pernah kejadian di sini.

**Kirim frame harus di-serialisasi.** `NS.kirim_aman()` ngunci proses
ambil-counter → enkripsi → kirim jadi satu blok. Wajib lewat fungsi ini, jangan
panggil `kirim_frame` + `ws_kirim` sendiri-sendiri dari lebih dari satu
goroutine. Di Nusantara, panggilan plugin native ngelepas GIL, jadi dua
goroutine bisa nyelak di tengah `kirim_frame` dan ngirim frame dengan urutan
counter kebalik. Server nolak dengan MAC failure terus nutup koneksi (WebSocket
close 1011).

**`receipt` dan `call` wajib di-ACK.** Bukan cuma `message` sama `notification`.
WhatsApp pakai flow control: dia kirim sebatch, nunggu ACK, baru lanjut. Kalau
`receipt` gak di-ACK, antrian offline mandek total — server bilang punya 24
pesan lewat `<ib><offline_preview>` tapi gak ngirim satupun. Pakai
`MS.buat_node_ack(node)` buat semua tag itu.

**Selama nunggu respons IQ tertentu, node lain jangan dibuang.** Alur kayak
fetch group-info / usync / prekey-bundle itu nunggu satu `<iq>` dengan `id`
tertentu. Node yang datang di tengah-tengah (pesan masuk, receipt, notification)
harus tetep dilempar ke `proses_node`, bukan di-skip — kalau di-skip, pesan itu
hilang diem-diem dan gak ada jejaknya di log.

**`pkmsg` dipakai terus sampai penerima bales, bukan cuma sekali.** Ini niru
`SessionCipher.HasUnacknowledgedPreKeyMessage()` di libsignal. Parameter
pembungkusnya (`base_pub`, `their_spk_id`, `otp_id`) disimpen di tabel
`send_sessions`, jadi gak perlu fetch prekey ulang tiap kirim. Kolom `acked`
baru jadi `1` setelah device itu beneran ngirim pesan ke kita; sesudah itu baru
pindah ke `type="msg"`. Kalau pindah ke `msg` kecepetan, penerima yang belum
sempet proses `pkmsg` pertama gak akan pernah bisa decrypt.

**Retry itu retransmisi, bukan pesan baru.** Pas dapet `<receipt type="retry">`,
kirim ulang pakai `msg_id` dan `t` yang **sama**, cuma ke device yang minta
(atribut `participant`), pakai `MS.buat_node_retry_kirim`. Kalau bikin `msg_id`
baru, WhatsApp nganggep itu pesan baru dan muncul dobel di chat — terus penerima
minta retry lagi buat id baru itu, jadi loop.

**`ws_terima` cuma boleh dipanggil dari SATU thread** — biasanya loop koneksi
utama kamu sendiri (bukan bagian dari `client.ns`, tapi pola yang wajib
diikutin). Native plugin `ws.cpp` gak ngunci buffer baca (`rbuf`) sama sekali,
cuma buffer tulis yang dikunci (`wmu`) — dua pemanggil `ws_terima` bareng bisa
korup buffer itu di level native, bukan cuma race biasa. Kalau butuh nunggu
balesan IQ tertentu (`tunggu_iq`) dari kode yang jalan di goroutine lain,
serahin ke loop utama lewat mekanisme kanal per-request (lihat
`coba_serahkan_ke_penunggu`/`tunggu_iq` di `client.ns`), jangan bikin goroutine
lain ikut manggil `ws_terima`.

**`tunggu_iq` gak boleh dipanggil sinkron dari loop koneksi utama sendiri.**
Fungsi yang butuh `tunggu_iq` (fetch device list, prekey bundle, media_conn,
dst) harus dijalanin di goroutine (`jalan(...)`), karena balesannya cuma bisa
diisi sama loop utama pas dia manggil `ws_terima` lagi. Kalau dipanggil
langsung dari loop utama, dia nunggu dirinya sendiri — deadlock, lepas cuma
setelah timeout internal `tunggu_iq` (default 30 detik), dan selama itu SEMUA
koneksi/chat kebekukan, bukan cuma satu request.

**Channel (`kanal_baru`/`kanal_kirim`/`kanal_terima`) + GC bisa deadlock kalau
interpreter-nya versi lama.** Ada bug lock-ordering di interpreter Nusantara
(`kanal_kirim`/`kanal_terima` ngunci `chan->mu` duluan baru daftar root ke GC,
sedangkan `GC::collectNow()` ngunci `gcMutex_` duluan baru butuh `chan->mu` pas
nge-scan isi channel) yang bisa bikin SEMUA thread nyangkut bareng kalau GC
kebetulan collect pas ada goroutine lagi di tengah operasi channel. Udah
di-fix di source interpreter (2026-08-31) — pastiin binary `nusa` yang dipake
udah versi yang ngandung fix ini kalau bikin bot baru dari nol pake
hypermeow-ns dan ngalamin freeze total yang gak jelas sebabnya (semua thread
`futex_do_wait` di gdb, restart count gak nambah = bukan crash).

## Media

Semua kripto media dikerjain di `.ns`. Transfer HTTP-nya lewat `curl` karena
plugin `http` bawaan ngasih body-nya lewat JSON, yang gak aman buat data biner.
`baca_file`/`tulis_file` sendiri udah binary-safe.

Unduh:

```
buat mc_node = MEDIA.buat_node_media_conn(req_id);   // kalau butuh host
buat isi = MEDIA.unduh_media(url, media_key, "image", sha256_hex, "/tmp/x.enc");
tulis_file("hasil.jpg", isi);
```

Unggah:

```
buat siap = MEDIA.siapkan_upload(baca_file("foto.jpg"), "image");
buat mc   = MEDIA.urai_media_conn(respons_iq);
buat up   = MEDIA.unggah_media(siap, "image", mc, "/tmp/x.enc");
// up["url"], up["direct_path"], up["media_key"], up["file_sha256"],
// up["file_enc_sha256"], up["file_length"] -> masukin ke protobuf pesan
```

Tipe yang didukung: `image`, `video`, `audio`, `ptt`, `document`, `sticker`.

## Reaksi & polling

Dua-duanya pakai "message secret" — kunci 32 byte yang ikut di
`messageContextInfo` (field 35) pesan aslinya. Tanpa nyimpen kunci itu waktu
pesan asli masuk, reaksi/vote yang datang belakangan gak bisa dibuka.
`client.ns` udah otomatis nyimpen lewat `SEC.simpan_secret_dari_pesan`.

```
buat r  = SEC.bangun_reaksi(chat, pengirim, msg_id, salah, "👍");
buat rh = SEC.bangun_hapus_reaksi(chat, pengirim, msg_id, salah);

buat p = SEC.bangun_poll("Makan apa?", ["Nasi", "Mie"], 1);
buat v = SEC.bangun_vote_poll(chat, kita, pembuat_poll, poll_id, salah, ["Mie"]);
```

Vote yang masuk cuma berisi SHA256 tiap opsi, bukan teksnya. Cocokkan pakai
`SEC.cocokkan_hash_opsi(hash_list, nama_opsi)` dengan daftar opsi dari poll
aslinya.

Turunan kuncinya: `HKDF-SHA256(secret, info = msg_id + pengirim_asli +
pengirim_modifikasi + tipe, 32)`, lalu AES-GCM. Khusus poll vote, AAD-nya
`msg_id + "\0" + pengirim_modifikasi`; buat reaksi AAD-nya kosong. Salah satu
aja beda, dekripsinya gagal total.

## App state

Sync kontak/arsip/bisu/sematan/bintang/label. Server ngirim "patch" yang tiap
mutasinya dienkripsi pakai kunci turunan dari app-state key yang dikirim
perangkat utama lewat pesan `protocolMessage`.

Integritasnya pakai **LTHash** — hash homomorfik 128 byte: nambah item itu
penjumlahan pointwise uint16 little-endian, ngapus item itu pengurangan. Jadi
state bisa maju-mundur tanpa ngitung ulang dari nol. Kalau salah endian atau
salah lebar, MAC snapshot-nya gak akan pernah cocok.

```
buat n = AS.buat_node_ambil_patch(req_id, AS.NAMA_REGULAR, versi_terakhir, salah);
buat kol = AS.urai_koleksi_patch(respons);
untuk (buat i = 0; i < panjang(kol["patch"]); i = i + 1) {
    buat h = AS.proses_patch(kol["nama"], kol["patch"][i], hash_sekarang, ambil_kunci);
    untuk (buat j = 0; j < panjang(h["mutasi"]); j = j + 1) {
        cetak(AS.ringkas_mutasi(h["mutasi"][j]));
    }
}
```

`ambil_kunci(key_id)` harus balikin hasil `AK.expand_app_state_keys(key_data)`
buat key_id itu. Kunci-nya disimpen di tabel `appstate_keys`, versi & hash
terakhir di `appstate_version`.

## Status

Jalan: pairing kode 8 digit, kirim/terima DM, kirim/terima pesan grup (teks,
gambar, video, dokumen, kutipan), retry receipt dua arah, auto top-up prekey,
reconnect otomatis, kelola grup, query user, presence, unduh/unggah media,
reaksi & polling, notifikasi grup, tolak panggilan otomatis, channel, app
state.

Belum: fitur business/katalog (`business.go`, ~3000 baris), argo/MEX decoding
buat sebagian respons channel, pairing lewat passkey.
