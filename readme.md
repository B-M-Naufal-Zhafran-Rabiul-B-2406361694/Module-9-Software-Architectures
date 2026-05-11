# Module 9 — Tutorial A: Event-Driven Architecture (Subscriber)

Repository ini berisi kode **subscriber** untuk tutorial Event-Driven Architecture menggunakan Rust, RabbitMQ, dan library `crosstown_bus`.

## Jawaban Pertanyaan

### a. What is *amqp*?

**AMQP** (Advanced Message Queuing Protocol) adalah *open standard application-layer protocol* untuk *message-oriented middleware*. Protokol ini mengatur bagaimana pesan dikirim, di-route, di-antri, dan di-deliver antar aplikasi yang saling terpisah, sehingga komunikasi tetap *interoperable* meskipun aplikasi-aplikasi tersebut ditulis dalam bahasa pemrograman atau berjalan di platform yang berbeda.

Karakteristik utama AMQP:
- *Message-oriented*: komunikasi dilakukan melalui pengiriman pesan, bukan pemanggilan fungsi langsung.
- *Queuing*: pesan disimpan di antrian sampai konsumer siap memprosesnya, sehingga *producer* dan *consumer* tidak harus aktif pada saat yang sama (*decoupling*).
- *Routing*: broker dapat mengarahkan pesan berdasarkan aturan tertentu (exchange, routing key, binding).
- *Reliability*: mendukung *acknowledgement*, *durable queue*, dan *dead-letter queue* agar pesan tidak hilang.
- *Security*: mendukung autentikasi (SASL) dan enkripsi (TLS).

**RabbitMQ** adalah salah satu *message broker* paling populer yang mengimplementasikan protokol AMQP (versi 0-9-1). Pada kode tutorial ini, URI `amqp://...` adalah skema URI standar yang dipakai untuk membuka koneksi AMQP ke broker.

### b. Arti dari `guest:guest@localhost:5672`

URI lengkap `amqp://guest:guest@localhost:5672` mengikuti format URI AMQP:

```
amqp://<username>:<password>@<host>:<port>
```

Pemecahan per komponen:

| Komponen | Nilai | Arti |
|----------|-------|------|
| Skema | `amqp` | Protokol yang digunakan (AMQP, terhubung ke RabbitMQ). |
| **`guest` (pertama)** | `guest` | **Username** yang dipakai untuk autentikasi ke broker RabbitMQ. `guest` adalah user default bawaan RabbitMQ. |
| **`guest` (kedua)** | `guest` | **Password** untuk user di atas. Password default user `guest` di RabbitMQ juga `guest`. |
| **`localhost`** | `localhost` | **Host / alamat server** tempat RabbitMQ berjalan. `localhost` (`127.0.0.1`) berarti broker dijalankan di mesin yang sama dengan aplikasi (dalam tutorial ini lewat Docker yang men-*expose* port-nya ke host lokal). |
| **`5672`** | `5672` | **Port AMQP default** RabbitMQ untuk koneksi non-TLS. (Port `5671` untuk AMQPS/TLS, dan `15672` untuk web management UI, jangan tertukar.) |

Catatan keamanan: user `guest/guest` adalah kredensial default RabbitMQ dan secara *default* hanya bisa login dari `localhost`. Untuk lingkungan produksi, user `guest` sebaiknya dihapus atau di-disable dan diganti dengan user khusus berikut password yang kuat.

## Simulating Slow Subscriber

Untuk meniru kondisi *demand* tinggi tapi processing lambat (mirip SIAK War), di `src/main.rs` baris `thread::sleep(ten_millis);` (variabel `ten_millis` berisi `Duration::from_millis(1000)`, jadi sebenarnya 1 detik per pesan) di-*uncomment*. Setelah `cargo build` dan `cargo run`, subscriber sekarang butuh ±1 detik untuk memproses tiap pesan.

Lalu publisher di-*fire* berkali-kali dengan cepat (`cargo run` berturut-turut di `publisher/`) supaya producer jauh lebih cepat daripada consumer. Hasilnya di RabbitMQ Management UI seperti berikut:

![Slow subscriber — antrian Ready menumpuk lalu turun pelan](slow-subscriber.png)

### Kenapa angka total queue bisa setinggi itu?

Pada eksekusi di mesin saya, baris **Ready** di chart *Queued messages* memuncak di sekitar **17 pesan** (di mesin instruktur 20), lalu turun **linear** ke 0 dalam ±20 detik. Penjelasannya:

- **Producer cepat, consumer lambat.** Saya menjalankan 8 instance publisher hampir bersamaan, total `8 × 5 = 40` pesan ter-*publish* ke queue `user_created` dalam waktu < 1 detik. Sementara itu subscriber hanya bisa memproses **1 pesan per detik** karena ada `thread::sleep(1000ms)` di handler.
- **Selisih rate inilah yang menumpuk di queue.** RabbitMQ menjamin pesan tidak hilang, jadi pesan yang belum sempat di-*deliver* ke consumer ditahan di queue dengan status **Ready** (siap dikirim tapi belum diambil). Saat momen ditangkap, broker sudah men-*deliver* sebagian (sudah jadi *Unacked* lalu di-*ack* lalu hilang dari hitungan), tapi masih ada ±17 pesan yang antri menunggu giliran.
- **Kenapa slope turunnya linear?** Karena subscriber memproses dengan rate konstan ±1 msg/detik (akibat `thread::sleep` yang fixed). Tiap detik, queue berkurang persis 1 pesan, sehingga garisnya turun lurus, bukan eksponensial.
- **Kenapa angkanya bukan 40?** Ada dua sebab: (1) saat saya membuka browser dan men-*screenshot*, sebagian pesan sudah keburu di-*consume* — kira-kira 20-an pesan sudah selesai sebelum tangkapan layar dibuat. (2) Chart *last minute* di RabbitMQ menampilkan **sample teragregasi** (titik per 5 detik), bukan nilai puncak sesaat, jadi angkanya tampak sedikit lebih halus daripada lonjakan sebenarnya.
- **Apa artinya bagi sistem nyata?** Tepat seperti analogi SIAK War: kalau backend (consumer) lebih lambat dari rate request, queue akan terus tumbuh. Tanpa message broker, kelebihan beban itu langsung memukul backend dan bisa membuatnya *crash*. Dengan broker di tengah, beban "diserap" sementara di queue, dan backend bisa menggerusnya pelan-pelan sesuai kecepatannya tanpa kehilangan pesan — *trade-off*-nya adalah *latency* pesan jadi naik.

## Running at Least Three Subscribers

Karena subscriber lambat (1 detik per pesan), solusinya adalah men-*scale out* — jalankan beberapa instance subscriber sekaligus, semuanya *consume* queue yang sama (`user_created`). Caranya: buka 3 terminal terpisah, di tiap terminal `cd subscriber && cargo run`, lalu *fire* publisher beberapa kali di terminal ke-4.

Hasil dari satu sesi `8 × cargo run publisher` (total 40 pesan) yang saya jalankan:

| Instance | Pesan diterima |
|---|---|
| Subscriber #1 | 14 |
| Subscriber #2 | 13 |
| Subscriber #3 | 13 |
| **Total** | **40** |

Distribusi nyaris seimbang — itu karena RabbitMQ secara default menggunakan algoritma **round-robin** untuk membagikan pesan ke semua consumer yang sedang aktif di queue yang sama. Setiap consumer dapat pesan secara bergantian, dan tidak ada satu pun pesan yang diproses dua kali (kecuali kasus *redelivery*).

Tampilan RabbitMQ Management UI saat 3 subscriber terhubung dan publisher di-*fire* berulang:

![Tiga subscriber paralel — queue cepat menyusut](three-subscribers.png)

### Kenapa spike queue lebih cepat redup dibanding sebelumnya?

- **Throughput menjadi N×.** Dengan 3 consumer yang masing-masing 1 msg/detik, total *deliver rate* efektif menjadi ±3 msg/detik. Untuk batch 40 pesan, drain selesai dalam ±13 detik — vs ±40 detik saat hanya 1 consumer.
- **Slope turunnya 3x lebih curam.** Bandingkan dengan grafik di section *Simulating Slow Subscriber*: di sana slope turun ±1 msg/detik, di sini ±3 msg/detik.
- **Bukti di Global counts:** `Connections: 3, Channels: 3, Consumers: 3` (sebelumnya 1/1/1) — RabbitMQ mendaftarkan ketiga subscriber sebagai consumer terpisah pada queue `user_created`.
- **Implikasi arsitektural.** Inilah kekuatan utama event-driven + message broker: *horizontal scaling* consumer **tidak perlu mengubah publisher sama sekali**. Kalau besok beban naik 10x, cukup tambah instance subscriber, broker otomatis membagi beban. Publisher tidak perlu tahu ada berapa subscriber.

### Refleksi: hal-hal yang bisa diperbaiki di kode

Beberapa observasi setelah menjalankan publisher & subscriber:

1. **Error pada `publish_event` di-*discard*.** Di [publisher/src/main.rs](https://github.com/B-M-Naufal-Zhafran-Rabiul-B-2406361694/Module-9-Software-Architectures-Publisher) tiap pemanggilan ditulis `_ = p.publish_event(...)`. Kalau koneksi ke broker putus saat pesan ke-3, kesalahannya hilang tanpa jejak. Lebih baik propagate via `?` (mengubah `fn main()` jadi `fn main() -> Result<(), Box<dyn Error>>`) atau minimal `eprintln!` log error sebelum dibuang.
2. **`UserCreatedEventMessage` di-*duplikasi* di dua repo.** Sekarang struct yang sama persis didefinisikan di subscriber **dan** publisher. Kalau ada satu field baru, dua repo wajib diubah konsisten — celah desync klasik. Idealnya tipe event dipindah ke *crate library* terpisah (`events-shared`) yang di-import oleh keduanya.
3. **`loop {}` di subscriber adalah *busy spin*.** Setelah `listener.listen(...)` mendaftar, thread utama hanya perlu menahan proses tetap hidup. `loop {}` membakar 100% satu core CPU tanpa tujuan. Cukup ganti dengan `std::thread::park();` atau menunggu sinyal Ctrl-C agar idle dengan benar.
4. **Tidak ada graceful shutdown.** Ctrl-C langsung memutus koneksi, padahal mungkin masih ada pesan yang sedang diproses (`unacked`). Idealnya tangkap sinyal SIGINT, hentikan penerimaan pesan baru, tunggu *handler* yang berjalan selesai, baru tutup channel.
5. **Konfigurasi *hardcoded*.** URL broker, nama queue, nilai *user_name* contoh, semua ditulis langsung di `main.rs`. Pindahkan ke *environment variable* atau argumen CLI agar mudah di-*deploy* ke staging/production tanpa rebuild.
6. **Library `crosstown_bus` di-*pin* ke versi lama.** Saya harus pin `crosstown_bus = "=0.5.3"` di subscriber dan `=0.5.0` di publisher karena versi `0.5.4` menambah method baru di trait `MessageHandler` yang membuat kode tutorial gagal compile. Library ini terlihat tidak aktif di-maintain — untuk proyek serius, pertimbangkan client AMQP modern Rust seperti **[lapin](https://crates.io/crates/lapin)** yang berbasis async/tokio dan komunitasnya jauh lebih besar.
7. **Variabel unused di handler.** `let now = time::Instant::now();` ditulis tapi tidak dipakai (mungkin sisa contoh pengukuran waktu). Hapus saja, atau kalau memang ingin mengukur lama proses tiap pesan, log selisihnya: `println!("processed in {:?}", now.elapsed())`.
8. **Tidak ada *publisher confirm* / persistence.** Pesan dipublish ke queue yang `durable: false` dan tanpa publisher confirm. Kalau broker restart, semua pesan in-flight hilang dan publisher juga tidak tahu bahwa pesan tidak benar-benar diterima. Untuk delivery garansi, ubah ke `durable: true` + aktifkan publisher confirm.

## Bonus: Simulating Slow Subscriber di Broker Cloud

Eksperimen *slow subscriber* diulang dengan broker dipindah dari Docker lokal ke **CloudAMQP** (managed RabbitMQ, plan gratis "Little Lemur" region AWS Asia Pacific). URL broker dibaca dari environment variable `AMQP_URL` dengan fallback ke localhost:

```rust
let amqp_url = std::env::var("AMQP_URL")
    .unwrap_or_else(|_| "amqp://guest:guest@localhost:5672".to_owned());
let listener = CrosstownBus::new_queue_listener(amqp_url).unwrap();
```

Cara jalankan:

```bash
AMQP_URL='amqps://USER:PASS@HOST/VHOST' cargo run
```

Skema `amqps://` (TLS port 5671) sudah dibuka di firewall CloudAMQP, jadi tidak ada konfigurasi tambahan dari sisi user.

Setelah subscriber terhubung ke cloud (dengan `thread::sleep` aktif = 1 detik per pesan), publisher di-*fire* 8x berturut-turut (40 pesan). Hasilnya di CloudAMQP Management UI:

![Slow subscriber via CloudAMQP — antrian Ready menumpuk lalu turun pelan](cloud-slow-subscriber.png)

Pola yang terlihat persis sama dengan eksperimen lokal: lonjakan Ready ke puluhan pesan, lalu turun linear ±1 msg/detik karena subscriber memang dibatasi `thread::sleep(1000ms)`. Perbedaan kentara dari versi lokal:

- URL bar = `armadillo.rmq.cloudamqp.com` (cloud), bukan `localhost`.
- Versi broker RabbitMQ `4.2.4` vs lokal `3.13`.
- Latensi tiap pesan jauh lebih besar karena hop ekstra mesin lokal → datacenter CloudAMQP → balik ke mesin lokal lewat TLS — tapi pola antrian *backpressure* tetap sama, broker tetap menahan pesan yang belum sempat di-consume.

Yang membuktikan poin event-driven architecture: **publisher dan subscriber sama sekali tidak diubah** untuk pindah dari Docker lokal ke broker cloud. Cukup ganti satu environment variable. Itulah keuntungan dari *decoupling* via message broker — lokasi fisik broker hanya detail deployment, bukan urusan kode aplikasi.
