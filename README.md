# ndlabs-bot

Self-bot WhatsApp ditulis di Nusantara (`.ns`), jalan di atas modul protokol [UwUchan](https://github.com/NusaLang/UwUchan).

## Pasang

```
nusa get github.com/NusaLang/UwUchan
cp .env.example .env
```

Isi `.env` (nomor bot, nomor owner), terus `nusa main.ns`.

## Arsitektur

```
main.ns               -> entry point, load session, konek WS
  bot/runtime.ns       -> loop koneksi, satu-satunya pemanggil ws_terima
  bot/bot.ns           -> HandleMessage: dispatch command, arsip, eval/shell
    plugin/manager.ns  -> registry plugin
    plugin/menu.ns     -> render .menu
    plugins/*.ns       -> satu file = satu/beberapa command
  nusantara_modules/UwUchan/
    client.ns          -> connect, decrypt, kirim_*_grup, buat_api
    msgsend.ns         -> bangun pesan / ekstrak budy & kutipan
    media.ns           -> upload/download media terenkripsi
    proto/pb.ns        -> encoder/decoder protobuf + skema named-field
    store/session.ns   -> SQLite sesi Signal, sender-key, device cache
```

Command masuk lewat `bot.ns:HandleMessage(sender, text, from_me, wa_snapshot)`. `wa_snapshot` dioper eksplisit, bukan baca `b["WA"]` belakangan, biar react/reply gak nyasar pas ada pesan lain numpuk barengan.

## Bahasa

`buat` (var), `fungsi`, `hasil` (return — jangan dipakai jadi nama variabel), `jika`/`lain`, `selama`, `untuk`, `coba`/`tangkap`, `lempar`, `lanjut`/`berhenti`. `kosong` = null, `benar`/`salah` = true/false, `peta_baru()` map diakses `["key"]` doang. `bentuk Nama { a, b }` buat struct, `kelas Nama { fungsi konstruktor(...) }` buat class beneran.

Fungsi top-level dobel nama gak error compile, diem-diem definisi terakhir yang menang. Sebelum deploy biasanya jalanin `grep -oE "^fungsi [a-zA-Z_0-9]+" file.ns | sort | uniq -d` buat ngecek.

## Plugin

```
buat BASE = impor("../plugin/plugin.ns");

fungsi NewNamaFiturPlugin() {
    buat p = BASE.baru_base_plugin("NamaFitur", "1.0.0", "ndlabs", "kategori", "deskripsi");
    p["GetCommands"] = fungsi() { hasil ["cmd", "alias1"]; };
    p["Execute"] = fungsi(m, args) {
        buat wa = m["WA"];
        wa.KirimTeks("halo");
        hasil "";
    };
    hasil p;
}
```

`m["WA"]` isinya method kirim (`KirimTeks`, `KirimGambar`, `KirimVideo`, `KirimAudio`, `KirimStiker`, `KirimPoll`, `KirimLokasi`, `KirimInteraktif` buat tombol native_flow, plus info grup kayak `GrupInfo`/`GrupPesertaDetail`). `m["budy"]` isinya body/tipe pesan, media, sama kutipan kalau lagi reply sesuatu.

`m["Reply"](teks)` cuma nge-log doang, gak beneran ngirim — yang kekirim itu nilai `hasil` dari `Execute`. Jadi kalau media udah dikirim manual pake `wa.KirimGambar(...)` dkk, jangan `hasil` string status kayak `"ok"` lagi, ntar nongol jadi teks nyasar di chat.

Kategori yang dikenal `plugin/menu.ns`: `main`, `grup`, `media`, `test`, `debug`, `owner`, sisanya masuk `lain`. Daftarin plugin baru di `bot/bot.ns` (import + `manager["Register"](...)`), terus `nusa -c` sebelum restart.

## Decode protobuf

`PB.pb_dekode_bernama(wire)` decode pake skema di `UwUchan/proto/pb.ns` yang di-generate dari proto whatsmeow lewat `UwUchan/tools/gen_pb_schema.py`. Field yang belum ada di skema fallback ke tag angka mentah. Decode skema selain `"Message"` panggil `_pb_dekode_bernama_inner(data, "NamaSkema", _pb_skema())` langsung.

## Concurrency

Cuma loop di `bot/runtime.ns` yang boleh manggil `ws_terima`. Kalau ada kode yang butuh `tunggu_iq(...)`, itu harus jalan di goroutine (`jalan(...)`), bukan sync langsung dari loop utama — kalau sync, loop utama nunggu balesan yang cuma bisa diisi sama dirinya sendiri, jadinya semua chat freeze sampai timeout.

`k["kirim_lock"]` ngejaga urutan kirim tetap sequential, tiap fungsi kirim wajib `kanal_kirim(k["kirim_lock"], 1)` di semua jalur keluar termasuk `tangkap` — kelewat satu, semua kirim berikutnya nyangkut.

Subprocess (`curl`/`ffmpeg` lewat `jalankan_perintah`) selalu dikasih batas waktu, curl pake `--connect-timeout`/`--max-time`, lainnya dibungkus `timeout -s KILL N`.

## Gotcha lain

`hasil` itu keyword, `buat hasil = ...` bakal error. Plugin native (`muat_plugin`) cuma nerima null/boolean/angka/teks, larik/peta kudu `json_encode` dulu. Tombol `ButtonsMessage` lama sering ditolak WhatsApp modern, pakai `KirimInteraktif` aja.
