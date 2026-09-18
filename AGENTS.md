# ndlabs-bot — panduan buat AI agent / kontributor

Self-bot WhatsApp ditulis di Nusantara (`.ns`), jalan di atas modul protokol WhatsApp+Signal custom (`nusantara_modules/hypermeow-ns`) — bukan whatsmeow/Baileys/wa-js. Dokumen ini buat siapa aja (manusia atau AI agent) yang mau nambah/ubah fitur di repo ini, atau bikin bot lain yang pakai `hypermeow-ns` sebagai modul protokol.

## Arsitektur singkat

```
main.ns / .main.ns   -> entry point, load session, konek WS, import semua modul
  bot/runtime.ns      -> loop koneksi utama: SATU-SATUNYA pemanggil ws_terima,
                          dispatch node masuk, sweep IQ waiter timeout, keepalive
  bot/bot.ns          -> HandleMessage: dispatch command, arsip, mute-check, eval/shell
    plugin/manager.ns -> registry semua plugin (Register/GetCommand/GetAll)
    plugin/menu.ns    -> render .menu (list+tombol interaktif)
    plugins/*.ns      -> satu file = satu/beberapa command
  nusantara_modules/hypermeow-ns/
    client.ns         -> WA protocol client: connect, decrypt, kirim_*_grup, buat_api (wa-api)
    msgsend.ns        -> bangun_pesan_*/ekstrak_budy/ekstrak_kutipan (protobuf message builder+parser)
    media.ns          -> upload/download media terenkripsi
    proto/pb.ns       -> encoder/decoder protobuf generik + skema named-field (_pb_skema, 916 message
                          type + 283 enum dari SEMUA 55 file .proto whatsmeow, auto-generated)
    socket/noisesocket.ns -> kirim_aman (frame terenkripsi Noise Protocol lewat WS)
    store/session.ns  -> SQLite: sesi Signal (kirim/terima), sender-key, device cache
```

Command masuk lewat `bot.ns:HandleMessage(sender, text, from_me, wa_snapshot)`, dipanggil dari `client.ns` tiap ada pesan grup/1:1 yang berhasil didekripsi. `wa_snapshot` **wajib** dioper eksplisit (bukan baca `b["WA"]` belakangan) — lihat bagian concurrency di bawah, ini nyegah react/reply nyasar ke pesan lain.

## Bahasa Nusantara — hal penting yang sering nge-trap

- Kata kunci: `buat` (deklarasi var), `fungsi`, `hasil` (return — **reserved, jangan dipakai jadi nama variabel**), `jika`/`lain`, `selama` (while), `untuk` (for), `coba`/`tangkap` (try/catch), `lempar` (throw), `lanjut` (continue), `berhenti` (break).
- `jenis` juga reserved (kepake internal) — jangan dipakai jadi nama parameter/variabel.
- Nilai: `kosong` (null), `benar`/`salah` (true/false), `peta_baru()` (map, akses `["key"]` doang, gak ada dot-notation buat map biasa).
- `bentuk Nama { field1, field2 }` — struct beneran, constructor positional (`Nama(v1, v2)`), akses field pakai titik (`x.field1`).
- `kelas Nama { fungsi method() {...} }` — class beneran, `fungsi konstruktor(...)`, akses method pakai titik.
- Array literal (`[a, b, c]`) itu `VmArray` (representasi VM yang dioptimasi), bukan `Array` lama — builtin yang nerima larik harus nge-cek dua-duanya (lihat gotcha `jalankan_perintah` di bawah kalau nulis builtin baru di level interpreter).
- String: `gabung(larik, sep)` (join), `pisah(s, sep)` (split), `potong(s, awal, akhir)` (slice), `panjang(s)`, `byte_di(s, i)`, `teks_dari(kode_byte)`.
- **Fungsi top-level dobel nama = diem-diem definisi terakhir yang menang, gak ada error compile.** Selalu `grep -oE "^fungsi [a-zA-Z_0-9]+" file.ns | sort | uniq -d` sebelum deploy.
- Gak pakai komentar di kode `.ns` kecuali beneran perlu jelasin alasan non-obvious (bukan APA yang dikerjain kode, tapi KENAPA — constraint tersembunyi, workaround bug spesifik).

## Objek pesan (`m`) yang diterima plugin

Tiap `p["Execute"] = fungsi(m, args) { ... }` menerima:

- `m["WA"]` — alias `wa`, wa-api buat chat saat ini (lihat daftar method di bawah). Ini **snapshot** yang dikunci pas pesan ini masuk, aman dipakai kapan aja selama eksekusi plugin walau ada pesan lain numpuk barengan.
- `m["Text"]` — teks lengkap pesan (termasuk prefix `.command`)
- `m["Sender"]` — JID pengirim (string)
- `m["Reply"](teks)` — **cuma nge-log, GAK beneran ngirim apa-apa.** Balasan asli itu nilai yang di-`hasil`-kan dari `Execute` — kalau non-kosong, otomatis dikirim sebagai teks ke chat. Jangan pernah `hasil` string status generik (`"ok"`, `"terkirim"`) kalau media/pesan lain udah dikirim manual di dalam fungsi — itu bakal nongol jadi spam teks aneh di chat. `hasil ""` kalau gak perlu balasan teks tambahan.
- `m["react"](emoji)` — kirim reaksi ke pesan yang lagi diproses (pola umum: `⏳` di awal, `✅`/`❌` di akhir buat command yang makan waktu)
- `m["budy"]` — hasil `MS.ekstrak_budy(fields)`, isi:
  - `budy["body"]` (teks), `budy["tipe"]` (`"conversation"`/`"extendedText"`/`"image"`/`"video"`/`"audio"`/`"document"`/`"sticker"`/`"reaction"`/dll — lihat `msgsend.ns:tipe_pesan`)
  - `budy["ada_media"]` (bool) + `budy["media"]` (map: `tipe_media`, `mimetype`, `file_length`, `caption`, `nama_file`, `direct_path`, `media_key`, dll — siap dipakai `wa["UnduhMedia"](media, path_tujuan)`)
  - `budy["kutipan"]` — kalau reply pesan lain: `stanza_id`, `participant`, `teks`, `pesan_wire` (bytes wire mentah pesan yang di-reply, buat `PB.pb_dekode_bernama`), `ada_media`+`media` (sama pola)
  - `budy["wire_mentah"]` — bytes wire mentah pesan yang lagi diproses
- `wa["konteks"]` — `chat_user`, `server`, `msg_id`, `ts`, `push_name`, `pengirim` (map `user`/`server`), `dikutip` (sama isi dgn `budy["kutipan"]`, referensi objek yang sama)
- `wa["IsGrup"]` (bool, bukan fungsi)

## wa-api (`m["WA"]`) — daftar method kirim

Semua kirim ke chat AKTIF (grup saat ini), kecuali disebut lain:

| Method | Guna |
|---|---|
| `KirimTeks(teks)` / `KirimKutip(teks)` | teks biasa / teks ngutip pesan yang diproses |
| `KirimStiker(path, crop, tipe_media_asli)` | convert gambar/video jadi stiker (ffmpeg); `tipe_media_asli` = `"image"` (statis) atau `"video"` (animasi, maks ~10.5 detik) |
| `KirimGambar(path, caption)` / `KirimGambarKutip` | upload gambar |
| `KirimVideo(path, caption)` / `KirimVideoKutip` | upload video |
| `KirimAudio(path, mimetype, ptt)` / `KirimAudioKutip` | upload audio, `ptt=benar` buat voice note |
| `KirimDokumen(path, nama_file, mimetype)` / `KirimDokumenKutip` | upload dokumen/file apapun |
| `KirimAlbum(items)` | `items` = larik peta `{path, tipe: "image"\|"video", caption}` — kirim 1 `AlbumMessage` placeholder + tiap item nempel `messageContextInfo.messageAssociation` balik ke situ, biar WA client ngegabung jadi tampilan album. **Eksperimental**, belum banyak dites lawan berbagai versi client. |
| `KirimReaksi(emoji)` / alias `react(emoji)` | reaksi ke pesan yang diproses |
| `KirimTeksExt(teks)` | extendedTextMessage polos |
| `KirimTeksAd(teks, judul, isi_kecil, path_thumb, besar)` | ad-reply card (thumbnail JPEG di-embed langsung, gak upload) — buat pengumuman/link promosi, **bukan** buat preview link |
| `KirimPreview(teks, url, judul, deskripsi, path_thumb, lebar, tinggi, besar)` | native link-preview (matchedText+title+description), thumbnail di-embed |
| `KirimPreviewMedia(teks, url, judul, deskripsi, path_thumb, lebar, tinggi)` | sama kayak `KirimPreview` tapi thumbnail-nya di-upload sbg media asli (dipake `.play`) |
| `KirimInteraktif(teks, tombol_list, footer, path_header_gambar)` | pesan interaktif modern (native_flow): tombol dari `TombolUrl`/`TombolBalasanCepat`/`TombolList`, header gambar di-upload |
| `KirimTombolLokasi(judul, alamat, isi, footer, tombol_list, path_thumb)` | `ButtonsMessage` legacy dgn header `LocationMessage` bawa thumbnail mentah (gak upload) — **WhatsApp modern seringnya nolak/gak render ini, dites dulu sebelum dipake serius** |
| `TombolUrl(label, url)` / `TombolBalasanCepat(label, id_perintah)` / `TombolList(label, sections)` | builder tombol native_flow buat `KirimInteraktif`/`KirimTombolLokasi` |
| `KirimPoll(nama, opsi, jml)` | poll |
| `KirimLokasi(nama, alamat, thumb)` | share lokasi |
| `UnduhMedia(media_map, path_tujuan)` | download+decrypt media (dari `budy["media"]` atau `budy["kutipan"]["media"]`) |
| `KirimWireMentah(wire)` | kirim protobuf `Message` mentah apa adanya (dipake `.sendraw`) |
| `SendMessage(message_struct, opsi)` | kirim struct proto `Message` penuh apa adanya (field sama kayak proto: `conversation`/`imageMessage`/dst) — `opsi["quoted"]`/`opsi["additionalNodes"]`/`opsi["label"]` opsional. Ini jalur paling fleksibel kalau method di atas gak cukup. |
| `KirimTeksKe(target_user, teks)` | kirim 1:1 (bukan grup) |
| `GrupInfo()` / `GrupPesertaDetail()` / `GrupPesertaDetailJID(group_user)` / `GrupAkuAdmin()` | info grup + daftar member (`admin`, `super_admin`, `pn_user`, `lid_user`) |
| `GrupSetNama`/`GrupSetDesk`/`GrupKunci`/`GrupAnnounce`/`GrupPeserta`/dll | admin action grup |

Reply/quote: kebanyakan punya varian `...Kutip` yang otomatis ngutip pesan yang lagi diproses (`wa["konteks"]["kutip"]`).

## Tipe pesan WhatsApp yang udah dikenal

`PB.pb_dekode_bernama(wire)` decode protobuf `Message` pake skema di `nusantara_modules/hypermeow-ns/proto/pb.ns:_pb_skema()` — **916 message type + 283 enum, dari SEMUA 55 file `.proto` whatsmeow** (waE2E, waCommon,  waArmadillo*/Instamadillo*/waCert/dll), termasuk resolusi referensi silang ANTAR file (mis. `ContextInfo` yang punya field nunjuk ke tipe di `waAICommon`, `waAea`, `waServerSync`, dst — bukan cuma yang satu file). Tipe dari `waE2E`/`waCommon` namanya polos tanpa prefix (`"Message"`, `"ImageMessage"`) demi backward-compat; tipe dari 53 file lain di-prefix nama direktori proto-nya, mis. `waAdv_ADVDeviceIdentity`, `waCert_NoiseCertificate`. Field yang **masih belum** ada di skema (proto baru yang belum dirilis whatsmeow, dll) otomatis fallback ke tag angka mentah. Generator-nya ada di `~/.claude/jobs/8963a5e0/tmp/gen_pb_schema.py` kalau perlu generate ulang pas whatsmeow update field baru. Buat decode skema SELAIN `"Message"`, panggil `_pb_dekode_bernama_inner(data, "NamaSkema", _pb_skema())` langsung (`pb_dekode_bernama` publiknya di-hardcode ke `"Message"`). Cara cari field number manual (kalau proto whatsmeow-nya sendiri belum sempet di-generate-ulang): buka source Go/whatsmeow yang ada di `/home/nopal/Documents/golang_botwa/whatsmeow/proto/`, grep `protobuf:"bytes,<N>,opt,name=<namaField>"`.

Field boolean pakai `_fldb(nama)`, enum pakai `_flde(nama, peta_enum)` (angka->nama, otomatis bisa di-encode balik lewat `.sendraw`), bytes yang mestinya tampil base64 (signature/secret/certificate) pakai `_fldbin(nama)`, submessage pakai `_flds(nama, "NamaSkemaLain")`.

Kalau nambah field baru di raw protobuf builder (`bangun_pesan_gambar` dkk di `msgsend.ns` — ini beda dari `_pb_skema()`, dipakai buat jalur kirim media yang udah lama/proven), field number harus PERSIS match urutan di proto asli whatsmeow, dan field yang berupa submessage (`messageContextInfo`, dll) itu SIBLING dari field media di level `Message` wrapper, bukan nested di dalam `imageMessage`/`videoMessage` itu sendiri — lihat contoh `bangun_asosiasi_album` (field 35 = `Message.messageContextInfo`) buat pola yang bener.

## Nambah plugin baru

1. Buat `plugins/nama_fitur.ns`:
   ```
   buat BASE = impor("../plugin/plugin.ns");

   fungsi NewNamaFiturPlugin() {
       buat p = BASE.baru_base_plugin("NamaFitur", "1.0.0", "ndlabs", "kategori", "deskripsi singkat");
       p["GetCommands"] = fungsi() { hasil ["cmd", "alias1"]; };
       p["Execute"] = fungsi(m, args) {
           buat wa = m["WA"];
           ...
           hasil ""; // atau teks balasan
       };
       hasil p;
   }
   ```
   Kategori yang dikenal `plugin/menu.ns` (`URUTAN_KATEGORI`): `main`, `grup`, `media`, `test`, `debug`, `owner`, selain itu masuk `lain`.
   Owner-only: `p["SetOwnerOnly"](benar)`. Sembunyiin dari `.menu`: `p["SetHidden"](benar)`.

2. Daftarin di `bot/bot.ns`: tambah `buat XXX = impor("../plugins/nama_fitur.ns");` di atas, lalu `manager["Register"](XXX.NewNamaFiturPlugin());` di dalam fungsi setup plugin.

3. **Wajib sebelum deploy:**
   ```
   nusa -c plugins/nama_fitur.ns
   nusa -c bot/bot.ns
   grep -oE "^fungsi [a-zA-Z_0-9]+" <file yang diedit> | sort | uniq -d   # pastikan gak ada fungsi dobel
   ```
   Kalau ngedit file di `nusantara_modules/hypermeow-ns/`, copy juga ke mirror `~/hypermeow-ns/` dan `~/.local/bin/nusantara-plugins/` — interpreter load dari sana, bukan cuma dari repo ini.

4. Deploy: `pm2 restart ndlabs-bot`, tunggu ~8-10 detik, cek `pm2 jlist` (status/mem/restart count) dan `grep -i FATAL ~/.pm2/logs/ndlabs-bot-error.log`.

## Concurrency & deadlock — WAJIB DIBACA sebelum nyentuh jalur kirim/terima

Ini bagian paling gampang bikin bot freeze TOTAL (semua grup, semua chat, bukan cuma satu command yang gagal) kalau dilanggar. Semua ini kejadian beneran dan udah di-debug pakai `gdb -p <pid> -batch -ex "thread apply all bt"` live di production.

- **Satu-satunya pemanggil `ws_terima` adalah loop utama di `bot/runtime.ns`.** Siapapun (goroutine atau kode lain) yang perlu nunggu balesan IQ (device resolve/prekey-fetch/usync/media_conn/dst) lewat `tunggu_iq(k, req_id, batas)` HARUS dipanggil dari dalam goroutine (`jalan(...)`), **gak boleh** langsung dari loop utama sendiri — kalau dipanggil sinkron dari situ, loop utama nunggu balesan yang cuma bisa diisi sama dirinya sendiri = deadlock permanen sampai timeout internal (30 detik) baru lepas, dan selama itu SEMUA chat gak kebales.
- Pola aman: pisahin logic yang butuh `tunggu_iq` ke fungsi top-level terpisah, panggil pake `jalan(nama_fungsi, k, arg1, arg2, ...)` dari titik yang tadinya mau manggil langsung.
- `k["kirim_lock"]` (channel-based mutex) ngejamin urutan strict-sequential noise-socket counter buat SEMUA fungsi `kirim_*_grup`/`kirim_*_1_1`. Setiap fungsi itu WAJIB `kanal_kirim(k["kirim_lock"], 1)` di SEMUA jalur keluar (sukses maupun `tangkap`) — kalau lupa satu jalur exception, lock ketinggalan ke-lock selamanya dan SEMUA kirim berikutnya (dari chat manapun) bakal nyangkut nunggu lock itu.
- **Bug interpreter yang udah ke-fix** (2026-08-31, `~/ndlabs-lang/nusantaracpp`): `kanal_kirim`/`kanal_terima` dulu ngunci `chan->mu` duluan baru daftar ke GC (`ValueVectorRootGuard`/`DepthResetGuard`, butuh `gcMutex_`) — AB-BA deadlock lawan `GC::collectNow()` yang ngunci `gcMutex_` duluan baru butuh `chan->mu`. Udah dibalik urutannya di source. Kalau nemu freeze total lagi yang polanya "semua thread stuck di futex, restart_time gak nambah", curigain kelas bug yang sama dan nangkep backtrace pake gdb (butuh `ptrace_scope=0`, jangan pernah minta password sudo user buat ini — biar user sendiri yang jalanin).
- `jalankan_perintah("nama_bin", argv)` (subprocess: curl/ffmpeg/ffprobe/dll) **WAJIB dibatasi waktu**, jangan pernah biarin unbounded:
  - curl: tambahin `"--connect-timeout", "15", "--max-time", "N"` (N sesuai konteks: 20-30 buat request kecil/API call, 60-120 buat download/upload file besar).
  - ffmpeg/ffprobe/bash/command lain yang gak punya flag timeout sendiri: bungkus pake `jalankan_perintah("timeout", ["-s", "KILL", "N", "nama_bin", ...argv_asli])`.
  - Alasan: kalau subprocess-nya nyangkut network stall/hang gak jelas, dan itu dipanggil dari goroutine yang jalan concurrent sama command lain, itu bisa nambah beban/waktu tunggu buat semua orang — batas waktu eksplisit biar selalu ada jalan keluar (sukses/gagal) daripada nunggu tanpa batas.
- `k["media_conn"]` (token auth upload media ke server WA) di-cache tapi gak ada expiry-check eksplisit — kalau bot idle lama terus tiba-tiba ada yang kirim media, upload bisa gagal `401`. `unggah_berkas` di `client.ns` udah nangani ini (retry sekali dengan `k["media_conn"] = kosong` abis dapet error yang ngandung "401") — kalau bikin jalur upload media baru yang gak lewat `unggah_berkas`, pertimbangin pola yang sama.

## Gotcha lain-lain

- `hasil` itu keyword reserved, jangan dipakai jadi nama variabel (`buat hasil = ...` bakal syntax error).
- Plugin native (`muat_plugin(...)`) cuma nerima `null`/`boolean`/`angka`/`teks` sebagai argumen — larik/peta harus `json_encode(...)` dulu (khususnya plugin `sqlite`).
- `m["Reply"]` gak ngirim apa-apa (lihat bagian objek pesan di atas) — nilai balik `Execute` itu yang beneran terkirim kalau non-kosong.
- `ButtonsMessage`/`KirimTombolLokasi` itu protokol lama yang WhatsApp modern sering nolak render — jangan andalin buat fitur utama, `KirimInteraktif` (native_flow) yang stabil dipakai.

## Deploy checklist ringkas

```
nusa -c <file yang diedit>
grep -oE "^fungsi [a-zA-Z_0-9]+" <file> | sort | uniq -d
# kalau file di nusantara_modules/hypermeow-ns/:
cp <file> ~/hypermeow-ns/<file>
cp <file> ~/.local/bin/nusantara-plugins/<file>
pm2 restart ndlabs-bot
sleep 10
pm2 jlist   # cek status online, mem wajar, restart_time gak lompat
grep -i FATAL ~/.pm2/logs/ndlabs-bot-error.log | tail -5
```
