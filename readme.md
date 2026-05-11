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
