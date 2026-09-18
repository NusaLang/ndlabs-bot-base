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

## Arsitektur

```
main.ns               -> entry point, load session, konek WS, import semua modul
  bot/runtime.ns       -> loop koneksi utama: satu-satunya pemanggil ws_terima,
                          dispatch node masuk, sweep IQ waiter timeout, keepalive
  bot/bot.ns           -> HandleMessage: dispatch command, arsip, mute-check, eval/shell
    plugin/manager.ns  -> registry semua plugin (Register/GetCommand/GetAll)
    plugin/menu.ns     -> render .menu (list+tombol interaktif)
    plugins/*.ns       -> satu file = satu/beberapa command
  nusantara_modules/UwUchan/
    client.ns          -> WA protocol client: connect, decrypt, kirim_*_grup, buat_api
    msgsend.ns         -> bangun_pesan_*/ekstrak_budy/ekstrak_kutipan
    media.ns           -> upload/download media terenkripsi
    proto/pb.ns        -> encoder/decoder protobuf generik + skema named-field
    socket/noisesocket.ns -> kirim_aman (frame terenkripsi Noise Protocol lewat WS)
    store/session.ns   -> SQLite: sesi Signal, sender-key, device cache
```

Command masuk lewat `bot.ns:HandleMessage(sender, text, from_me, wa_snapshot)`. `wa_snapshot` wajib dioper eksplisit, bukan baca `b["WA"]` belakangan — lihat bagian concurrency di bawah.

## Bahasa Nusantara, hal yang sering nge-trap

- `buat` (var), `fungsi`, `hasil` (return, **reserved**, jangan dipakai jadi nama variabel), `jika`/`lain`, `selama` (while), `untuk` (for), `coba`/`tangkap` (try/catch), `lempar` (throw), `lanjut`/`berhenti` (continue/break). `jenis` juga reserved.
- `kosong` (null), `benar`/`salah` (true/false), `peta_baru()` (map, akses `["key"]`, gak ada dot-notation).
- `bentuk Nama { field1, field2 }` — struct, constructor positional, akses field pakai titik. `kelas Nama { fungsi konstruktor(...) ... }` — class beneran.
- String: `gabung`/`pisah`/`potong`/`panjang`/`byte_di`/`teks_dari`.
- Fungsi top-level dobel nama = diem-diem definisi terakhir yang menang, gak ada error compile. Cek pake `grep -oE "^fungsi [a-zA-Z_0-9]+" file.ns | sort | uniq -d` sebelum deploy.

## Objek pesan (`m`) di plugin

```
p["Execute"] = fungsi(m, args) {
    buat wa = m["WA"];       // wa-api buat chat ini, snapshot terkunci pas pesan masuk
    buat teks = m["Text"];   // teks lengkap termasuk prefix .command
    buat pengirim = m["Sender"];
    m["react"]("⏳");
    ...
    hasil "";                 // atau teks balasan -- lihat catatan Reply di bawah
};
```

`m["Reply"](teks)` cuma nge-log, **gak beneran ngirim**. Yang beneran terkirim itu nilai `hasil` dari `Execute` (kalau non-kosong, otomatis dikirim sebagai teks). Jangan `hasil` status generik kayak `"ok"` kalau media udah dikirim manual di dalam fungsi — itu nongol jadi spam teks aneh di chat.

`m["budy"]` (dari `MS.ekstrak_budy`): `body`/`tipe` (`"conversation"`/`"image"`/`"video"`/dll), `ada_media`+`media` (map siap pakai `wa["UnduhMedia"]`), `kutipan` (kalau reply pesan lain: `stanza_id`, `teks`, `pesan_wire`, `ada_media`+`media`).

## wa-api (`m["WA"]`)

Kirim teks/media ke chat aktif: `KirimTeks`, `KirimKutip`, `KirimGambar(path, caption)`, `KirimVideo`, `KirimAudio(path, mimetype, ptt)`, `KirimDokumen(path, nama, mimetype)`, `KirimStiker(path, crop, tipe_asli)` — masing-masing punya varian `...Kutip` buat ngutip pesan yang diproses.

Interaktif: `KirimReaksi(emoji)`/`react`, `KirimPoll(nama, opsi, jml)`, `KirimLokasi`, `KirimInteraktif(teks, tombol_list, footer, gambar)` pakai tombol dari `TombolUrl`/`TombolBalasanCepat`/`TombolList`.

Grup: `GrupInfo()`, `GrupPesertaDetail()`, `GrupAkuAdmin()`, `GrupSetNama`/`GrupSetDesk`/`GrupKunci`/`GrupPeserta`/dll.

Lain-lain: `KirimTeksKe(target, teks)` (1:1), `UnduhMedia(media_map, path)`, `KirimWireMentah(wire)`/`SendMessage(struct, opsi)` (jalur mentah kalau method di atas gak cukup).

## Decode protobuf mentah

`PB.pb_dekode_bernama(wire)` decode `Message` pakai skema di `UwUchan/proto/pb.ns:_pb_skema()` (auto-generated dari semua `.proto` whatsmeow, termasuk resolusi referensi antar file). Field yang belum ada di skema otomatis fallback ke tag angka mentah. Generator ada di `UwUchan/tools/gen_pb_schema.py <path/ke/whatsmeow/proto>` buat regenerate pas whatsmeow nambah field baru. Decode skema selain `"Message"`: `_pb_dekode_bernama_inner(data, "NamaSkema", _pb_skema())`.

## Nambah plugin baru

```
buat BASE = impor("../plugin/plugin.ns");

fungsi NewNamaFiturPlugin() {
    buat p = BASE.baru_base_plugin("NamaFitur", "1.0.0", "ndlabs", "kategori", "deskripsi singkat");
    p["GetCommands"] = fungsi() { hasil ["cmd", "alias1"]; };
    p["Execute"] = fungsi(m, args) {
        buat wa = m["WA"];
        ...
        hasil "";
    };
    hasil p;
}
```

Kategori yang dikenal `plugin/menu.ns`: `main`, `grup`, `media`, `test`, `debug`, `owner`, selain itu masuk `lain`. `p["SetOwnerOnly"](benar)` buat command khusus owner, `p["SetHidden"](benar)` buat sembunyiin dari `.menu`.

Daftarin di `bot/bot.ns`: import di atas, `manager["Register"](XXX.NewNamaFiturPlugin())` di dalam fungsi setup. Sebelum deploy: `nusa -c` file yang diedit + cek fungsi dobel nama (lihat di atas). Kalau ngedit file di `nusantara_modules/UwUchan/`, copy juga ke mirror lokal kalau ada (interpreter load dari sana, bukan dari repo).

## Concurrency, wajib dibaca sebelum ubah jalur kirim/terima

Pelanggaran di sini bikin bot freeze total (semua chat), bukan cuma satu command gagal.

- **Cuma loop utama (`bot/runtime.ns`) yang boleh manggil `ws_terima`.** Kode apapun yang butuh nunggu balesan IQ lewat `tunggu_iq(k, req_id, batas)` harus dipanggil dari goroutine (`jalan(...)`), gak boleh langsung dari loop utama — kalau sync, loop utama nunggu balesan yang cuma bisa diisi dirinya sendiri = deadlock sampai timeout.
- `k["kirim_lock"]` (channel-based mutex) ngejamin urutan noise-socket counter strict-sequential buat semua fungsi kirim. Wajib `kanal_kirim(k["kirim_lock"], 1)` di SEMUA jalur keluar (sukses maupun `tangkap`) — lupa satu jalur, lock ketinggalan ke-lock selamanya dan semua kirim berikutnya nyangkut.
- `jalankan_perintah` ke subprocess (curl/ffmpeg/dll) wajib dibatasi waktu: curl pakai `--connect-timeout`/`--max-time`, yang lain dibungkus `timeout -s KILL N`. Subprocess nyangkut tanpa batas nambah beban ke semua command lain yang jalan concurrent.
- `k["media_conn"]` (token upload media) gak ada expiry-check eksplisit — kalau bot idle lama, upload bisa gagal `401`. Retry sekali dengan `media_conn` di-reset abis error yang ngandung "401".

## Gotcha lain

- `hasil` reserved, `buat hasil = ...` syntax error.
- Plugin native (`muat_plugin`) cuma nerima `null`/`boolean`/`angka`/`teks` — larik/peta harus `json_encode` dulu.
- `ButtonsMessage`/tombol legacy sering ditolak render WhatsApp modern — pakai `KirimInteraktif` (native_flow).
