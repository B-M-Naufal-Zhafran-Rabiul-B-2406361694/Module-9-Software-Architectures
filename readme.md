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
