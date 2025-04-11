# Dokumentasi Teknis Aplikasi Pemesanan Janji Temu Dokter

## Pendahuluan

### Deskripsi Bisnis
Aplikasi ini adalah platform digital untuk memesan janji temu dokter, dilengkapi dengan fitur chat otomatis melalui WhatsApp Business API (via Wati.io), manajemen jadwal dokter, pemesanan slot waktu, dan analitik untuk staf administrasi. Setelah pemesanan dikonfirmasi, notifikasi otomatis dikirim ke WhatsApp pasien. Aplikasi menggunakan **MongoDB** untuk database, **Google Cloud Platform (GCP)** untuk infrastruktur, dan **Redis** untuk caching.

### Tujuan
- Menyediakan antarmuka yang intuitif untuk pasien memesan dan mengelola janji temu.
- Mengotomatiskan komunikasi via WhatsApp untuk konfirmasi, pengingat, dan pertanyaan umum.
- Memberikan dashboard analitik real-time untuk memahami tren pemesanan.
- Memastikan performa, skalabilitas, dan keamanan melalui infrastruktur GCP.

### Pemangku Kepentingan
- **Pasien**: Memesan janji dan menerima konfirmasi via WhatsApp.
- **Dokter**: Mengelola jadwal dan menerima notifikasi pemesanan.
- **Staf Administrasi**: Mengelola data dan mengakses analitik.
- **Tim IT**: Memelihara sistem dan integrasi Wati.io.

## Fitur Utama

| Fitur          | Deskripsi                                                                 |
|----------------|---------------------------------------------------------------------------|
| **Chat**       | Integrasi dengan WhatsApp Business API via Wati.io untuk chat otomatis, menangani pertanyaan umum, konfirmasi pemesanan, dan pengingat. |
| **Pemesanan**  | Pasien mencari dokter berdasarkan spesialisasi/lokasi, memesan slot waktu, serta membatalkan/menjadwalkan ulang janji. |
| **Jadwal**     | Manajemen slot waktu dokter dengan sinkronisasi kalender dan status ketersediaan. |
| **Analitik**   | Dashboard dengan laporan seperti jumlah pemesanan, tingkat pembatalan, dan interaksi WhatsApp. |
| **Server**     | Infrastruktur GCP (Compute Engine, Cloud Storage) untuk hosting, skalabilitas, dan keamanan. |

## Arsitektur Sistem

### Gambaran Umum
Aplikasi menggunakan arsitektur **microservices** untuk memisahkan fungsi (pemesanan, jadwal, chat, analitik) agar skalabel. Komponen utama:
- **Frontend**: Antarmuka web/mobile berbasis React.js atau Flutter.
- **Backend**: Logika bisnis dan API menggunakan Node.js dengan Express.
- **Database**: MongoDB untuk data fleksibel, Redis untuk caching.
- **Integrasi Eksternal**: WhatsApp Business API via Wati.io.
- **Server**: GCP Compute Engine untuk hosting, Cloud Storage untuk file.

### Diagram Arsitektur
```
[Pasien/Dokter] ↔ [Frontend: React.js/Flutter]
                         ↓
                [API Gateway: GCP Cloud Endpoints]
                         ↓
[Backend Services: Node.js]
   ↔ [Database: MongoDB Atlas]
   ↔ [Cache: Redis (Memorystore)]
   ↔ [WhatsApp API: Wati.io]
   ↔ [Analytics: BigQuery]
                         ↓
                [Server: GCP Compute Engine/Cloud Storage]
```

### Alur Data
1. Pasien login → Autentikasi via JWT → Akses daftar dokter (cached di Redis).
2. Pasien memesan slot → Validasi jadwal → Simpan ke MongoDB → Kirim konfirmasi via Wati.io ke WhatsApp.
3. Dokter menerima notifikasi → Perbarui jadwal di MongoDB.
4. Staf administrasi akses dashboard → Analitik dari MongoDB/BigQuery.

## Spesifikasi Teknis

### Frontend
- **Framework**: React.js (web), Flutter (mobile).
- **Fitur**:
  - Pencarian dokter (filter: spesialisasi, lokasi).
  - Formulir pemesanan dengan pemilih tanggal/jam.
  - Riwayat pemesanan dan status.
  - Chat terintegrasi untuk pertanyaan.

### Backend
- **Framework**: Node.js dengan Express.
- **API**: RESTful dengan endpoint utama:
  - `POST /auth/login`: Autentikasi pengguna.
  - `GET /doctors`: Daftar dokter (cached di Redis).
  - `POST /bookings`: Membuat pemesanan baru.
  - `GET /analytics`: Data untuk dashboard.
- **Autentikasi**: JWT untuk sesi aman.
- **Integrasi Wati.io**:
  - Endpoint: `POST /whatsapp/send` untuk konfirmasi/pengingat.
  - Webhook untuk pesan masuk dari pasien.

### Database
- **MongoDB Atlas** (di-host di GCP):
  - Koleksi:
    - `users`: `{ id, name, email, whatsapp, role }`.
    - `doctors`: `{ id, name, specialty, schedule }`.
    - `bookings`: `{ id, user_id, doctor_id, date, time, status }`.
    - `chat_logs`: `{ id, user_id, message, timestamp }`.
  - Indeks pada `doctors.specialty` dan `bookings.date` untuk query cepat.
- **Redis (GCP Memorystore)**:
  - Cache daftar dokter populer dan slot waktu.
  - Sesi pengguna untuk autentikasi cepat.
  - Contoh:
    ```javascript
    const redis = require('redis');
    const client = redis.createClient({ url: process.env.REDIS_URL });
    await client.setEx('doctors:popular', 3600, JSON.stringify(doctors));
    ```

### Integrasi WhatsApp API dengan Wati.io
- **Penyedia**: Wati.io ([Wati.io](https://www.wati.io/)).
- **Fitur**:
  - Chatbot untuk pertanyaan umum (misalnya, ketersediaan dokter).
  - Pesan konfirmasi: “Janji temu Anda dengan Dr. X pada 11/04/2025 pukul 10:00 telah dikonfirmasi.”
  - Pengingat: 24 jam sebelum janji.
  - Logging interaksi untuk analitik.
- **Konfigurasi**:
  - Daftar akun Wati.io, verifikasi nomor WhatsApp Business.
  - Buat template pesan (contoh: `booking_confirmation`).
  - Setup webhook untuk pesan masuk.
  - Kode pengiriman pesan:
    ```javascript
    const wati = require('wati-sdk');
    async function sendConfirmation(booking) {
      await wati.sendMessage({
        phone: booking.user.whatsapp,
        message: `Janji temu Anda dengan ${booking.doctor.name} pada ${booking.date} pukul ${booking.time} telah dikonfirmasi.`,
        apiKey: process.env.WATI_API_KEY
      });
    }
    ```

### Server
- **Hosting**: GCP Compute Engine untuk server aplikasi, Cloud Storage untuk file (misalnya, foto dokter).
- **Load Balancer**: GCP HTTP(S) Load Balancer untuk distribusi trafik.
- **Auto-scaling**: Skalakan instance berdasarkan CPU usage (>70%).
- **Keamanan**:
  - SSL/TLS via GCP Certificate Manager.
  - VPC firewall untuk perlindungan DDoS.
  - Enkripsi data di MongoDB Atlas menggunakan AES-256.

### Analitik
- **Engine**: GCP BigQuery untuk analisis log pemesanan dan chat.
- **Dashboard**: Visualisasi dengan Looker Studio atau Chart.js.
- **Metrik**:
  - Pemesanan per hari/minggu.
  - Tingkat pembatalan per dokter.
  - Interaksi WhatsApp (pesan masuk/keluar).
  - Waktu respons bot.

## Implementasi

### Tahapan Pengembangan
1. **Setup Lingkungan** (1 minggu):
   - Konfigurasi GCP (Compute Engine, Cloud Storage, MongoDB Atlas, Memorystore).
   - Install Node.js, React.js, dan dependensi.
2. **Backend** (3 minggu):
   - Bangun API untuk autentikasi, pemesanan, jadwal.
   - Integrasi Wati.io untuk WhatsApp.
3. **Frontend** (2 minggu):
   - Desain UI/UX untuk pencarian, pemesanan, riwayat.
   - Hubungkan API ke frontend.
4. **Analitik** (1 minggu):
   - Setup BigQuery dan Looker Studio.
   - Bangun dashboard dengan metrik utama.
5. **Pengujian** (2 minggu):
   - Unit test untuk API.
   - Integrasi test untuk pemesanan dan WhatsApp.
   - Stress test untuk 10.000 pengguna.

### Alur Pemesanan dan Konfirmasi WhatsApp
1. Pasien pilih slot → Kirim `POST /bookings`.
2. Backend validasi slot → Simpan ke MongoDB → Panggil Wati.io:
   ```javascript
   await wati.sendTemplateMessage({
     phone: user.whatsapp,
     template: 'booking_confirmation',
     params: { doctor: doctor.name, date: booking.date, time: booking.time }
   });
   ```
3. Pasien terima pesan: “Janji temu Anda dengan Dr. X pada 11/04/2025 pukul 10:00 telah dikonfirmasi.”
4. Pengingat 24 jam sebelumnya via cron job:
   ```javascript
   const cron = require('node-cron');
   cron.schedule('0 0 * * *', async () => {
     const bookings = await getTomorrowBookings();
     bookings.forEach(booking => sendReminder(booking));
   });
   ```

## Pengujian
- **Unit Test**:
  - API: Uji `POST /bookings` dengan data valid/invalid.
  - WhatsApp: Uji pesan via Wati.io dengan nomor test.
- **Integration Test**:
  - Alur pemesanan: Frontend → Backend → MongoDB → Wati.io.
  - Sinkronisasi jadwal dokter.
- **Performance Test**:
  - Simulasi 10.000 pengguna dengan Locust.
  - Target: Latency <2 detik, uptime 99.9%.
- **Security Test**:
  - Uji injeksi NoSQL pada MongoDB.
  - Uji kebocoran JWT.

## Pemeliharaan
- **Pembaruan**: Rilis fitur baru setiap 2 bulan via CI/CD (Cloud Build).
- **Monitoring**: GCP Cloud Monitoring untuk server, Wati.io untuk WhatsApp.
- **Backup**: Snapshot harian MongoDB Atlas, log chat ke Cloud Storage.
- **Dukungan**: Tim support 24/7, SLA respons 1 jam.

## Risiko dan Mitigasi
| Risiko                          | Mitigasi                                                                 |
|---------------------------------|-------------------------------------------------------------------------|
| Downtime saat trafik tinggi      | Gunakan GCP auto-scaling dan load balancer.                             |
| Gagal kirim pesan WhatsApp      | Fallback ke email, retry mechanism untuk Wati.io.                       |
| Kebocoran data pasien           | Enkripsi MongoDB, kepatuhan HIPAA, audit keamanan rutin.                |
| Slot jadwal bentrok             | Validasi real-time, lock MongoDB untuk update jadwal.                   |

## Integrasi Wati.io
- **Mengapa Wati.io**:
  - Integrasi mudah dengan Node.js.
  - Template pesan untuk konfirmasi/pengingat.
  - Dashboard analitik WhatsApp.
  - Uji coba gratis 7 hari.
- **Langkah**:
  1. Daftar di [Wati.io](https://www.wati.io/), verifikasi nomor WhatsApp Business.
  2. Buat template pesan (misalnya, `booking_confirmation`).
  3. Konfigurasi webhook untuk pesan masuk.
  4. Gunakan API key di backend.
- **Biaya**: Lihat [Wati.io Pricing](https://www.wati.io/pricing/).

## Lampiran
### Contoh Payload Wati.io
```json
{
  "phone": "+6281234567890",
  "message": "Janji temu Anda dengan Dr. John pada 11/04/2025 pukul 10:00 telah dikonfirmasi."
}
```

### Diagram Alur Pemesanan
```
Pasien → Pilih Slot → Backend Validasi → Simpan MongoDB → Wati.io Kirim Konfirmasi → Pasien Terima WhatsApp
```

### Dependensi
- Node.js, Express, MongoDB Atlas, Redis (Memorystore), React.js, GCP SDK, Wati.io SDK.

---

### Catatan Tambahan
- **Format Markdown**: Dokumentasi di atas sudah dalam format `.md`. Anda bisa menyalinnya ke file seperti `doctor-appointment-app.md` dan membukanya di editor seperti VS Code atau platform seperti GitHub untuk tampilan yang rapi.
- **File Fisik**: Jika Anda ingin saya membantu mengirimkan file .md (misalnya, via email atau cara lain), silakan beri tahu caranya, karena saat ini saya hanya bisa menyediakan teks di sini.
- **Penyesuaian**:
  - Jika Anda ingin diagram (misalnya, UML atau flowchart), saya bisa menjelaskan dalam teks atau kode Mermaid untuk Markdown:
    ```mermaid
    sequenceDiagram
        Pasien->>Frontend: Pilih slot
        Frontend->>Backend: POST /bookings
        Backend->>MongoDB: Simpan booking
        Backend->>Wati.io: Kirim konfirmasi
        Wati.io->>Pasien: Pesan WhatsApp
    ```
