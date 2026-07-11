# Relasi Tiap Role -> UAS Communication Protocol
### Use Case 3: Task/Order Tracking Workflow Kelompok 05

---

## 1. Daftar Role & Anggota

| Nama | Role | Fokus Kontribusi |
|---|---|---|
| Uswatun Hasanah | **Role 1** — API & Postman Tester | Menyiapkan dan menjalankan pengujian endpoint lewat Postman: health check, order flow (success/failure), dan reliability test. |
| Aryadi | **Role 2** — Protocol & Traffic Analyst | Capture traffic dengan Wireshark; menganalisis request sukses, gagal, order found, dan order not found. |
| Faza Qinthoro Putra Tsany | **Role 3** — Integration/Workflow Engineer | Membangun workflow n8n: validasi order, success branch, failure branch, serta status check order. |
| Nabiel Azzacky | **Role 4** — Documentation & Presenter Lead | Menyusun struktur laporan, mengompilasi evidence, merapikan narasi presentasi, dokumentasi, dan finalisasi file. |

---

## 2. Kenapa Role-Role Ini Saling Terhubung

Keempat role ini **tidak bekerja terpisah** — mereka mengikuti satu alur data yang sama (Postman → REST API / n8n Webhook → Validasi → Response), tapi masing-masing mengamati dan mendokumentasikan alur itu dari sudut pandang berbeda. Output satu role menjadi bahan baku (input) buat role berikutnya.

```
Role 1 (Postman)  →  mengirim request  →  Role 3 (n8n Workflow)
      │                                          │
      │  request/response yang sama              │  branch success/failure
      ▼  direkam sebagai traffic                  ▼  yang sama dijadikan evidence
Role 2 (Wireshark)  ←──────── traffic sama ────────┘
      │
      ▼
Role 4 (Dokumentasi)  ←── mengumpulkan evidence dari Role 1, 2, dan 3
```

---

## 3. Relasi Detail Antar Role

### 🔗 Role 1 → Role 3 (Postman Tester ↔ Workflow Engineer)
Role 1 mengirim request (POST order, GET status) ke endpoint yang **dibangun oleh Role 3** di n8n. Tanpa workflow Role 3 aktif dan endpoint-nya terdaftar dengan benar, request Role 1 tidak akan mendapat response apa pun (404 "not registered"). Sebaliknya, Role 3 butuh hasil pengujian Role 1 untuk membuktikan bahwa branch `TRUE_BRANCH_SUCCESS` dan `FALSE_BRANCH_ERROR` benar-benar bisa dipicu dari luar, bukan cuma jalan manual di editor n8n.

### 🔗 Role 1 → Role 2 (Postman Tester ↔ Traffic Analyst)
Setiap kali Role 1 menjalankan request di Postman, traffic-nya lewat jaringan lokal di port `8088` (backend) dan `5678` (n8n). Traffic inilah yang **ditangkap Role 2** lewat Wireshark dengan filter `tcp.port == 8088 || tcp.port == 5678`. Jadi skenario yang dirancang dan dijalankan Role 1 (success, failure, found, not found) adalah skenario yang sama persis yang dianalisis Role 2 di level paket jaringan.

### 🔗 Role 2 → Role 3 (Traffic Analyst ↔ Workflow Engineer)
Traffic yang ditangkap Role 2 memuat payload JSON dan response yang **dihasilkan oleh logic workflow Role 3** (validasi `amount > 0` dan `customer` tidak kosong). Evidence Wireshark jadi bukti independen bahwa keputusan branch di n8n (CONFIRMED/REJECTED) benar-benar terjadi di jaringan, bukan cuma diklaim di laporan.

### 🔗 Role 1 + Role 2 + Role 3 → Role 4 (Ketiganya ↔ Documentation Lead)
Role 4 tidak menghasilkan evidence baru, tapi **mengumpulkan dan merangkai** semua bukti dari tiga role sebelumnya:
- Screenshot Postman (dari Role 1)
- Screenshot Wireshark (dari Role 2)
- Screenshot n8n execution/workflow (dari Role 3)

menjadi satu narasi laporan yang koheren, termasuk menyusun Tabel 1 (endpoint), Tabel 2 (skenario pengujian), dan Tabel 3 (kontribusi anggota).

---

## 4. Titik Temu Teknis (Shared Artifacts)

Ini bagian yang secara konkret "menyatukan" kerja keempat role — resource/data yang sama dipakai lintas role:

| Artifact | Dipakai Role 1 | Dipakai Role 2 | Dipakai Role 3 | Dipakai Role 4 |
|---|:---:|:---:|:---:|:---:|
| `POST /webhook/commproto-p10` | ✅ mengirim | ✅ menangkap traffic | ✅ membangun endpoint | ✅ mendokumentasikan |
| `GET /webhook/order-status/:orderId` | ✅ mengirim | ✅ menangkap traffic | ✅ membangun endpoint | ✅ mendokumentasikan |
| `GET /api/health` | ✅ mengirim | — | ✅ membangun endpoint | ✅ mendokumentasikan |
| Payload JSON success/failure | ✅ merancang | ✅ menganalisis isi | ✅ memvalidasi | ✅ mengutip contoh |
| Status code (200, 201, 400, 404, 429) | ✅ memverifikasi | ✅ mengonfirmasi via traffic | ✅ menentukan branch | ✅ merangkum di Tabel 2 |

---

## 5. Ringkasan Alur End-to-End (Lintas Role)

1. **Role 1** menyusun request di Postman (payload valid & invalid) → kirim ke endpoint.
2. **Role 3** menerima request lewat webhook n8n → jalankan `Validate Payload` → pilih branch (`TRUE_BRANCH_SUCCESS` / `FALSE_BRANCH_ERROR`) → kirim balik response JSON.
3. **Role 2** menangkap seluruh proses request-response ini di level paket jaringan lewat Wireshark, sebagai bukti independen bahwa komunikasi memang terjadi lewat port 8088/5678.
4. **Role 4** mengumpulkan evidence dari ketiga role di atas (Postman, Wireshark, n8n execution) dan menyusunnya jadi laporan UAS yang utuh, termasuk bab Analisis, Evaluasi, dan Kesimpulan.

---

## 6. Refleksi Tiap Role

> Catatan: bagian ini disusun sebagai draft dengan bahasa sederhana, berdasarkan konteks kontribusi dan tantangan yang tersirat di laporan (BAB V–VII, IX). Silakan disesuaikan lagi dengan pengalaman pribadi masing-masing anggota sebelum dimasukkan ke laporan final.

### Role 1 — Uswatun Hasanah (API & Postman Tester)

**Apa yang dikerjakan**
Selama project ini, saya bertugas nge-test semua endpoint pakai Postman. Mulai dari cek server nyala (health check), kirim order yang benar (success), kirim order yang salah (failure), sampai cek status order yang ada dan yang tidak ada.

**Yang dipelajari**
- Testing itu bukan cuma "kirim request terus lihat hasilnya", tapi harus tahu juga *kenapa* hasilnya begitu. Misalnya, status code 200 itu artinya berhasil, tapi 404 belum tentu salah — bisa jadi memang itu hasil yang diharapkan (kayak pas cek order yang sengaja tidak ada).
- Ternyata endpoint yang kelihatan "sudah benar" di URL bisa tetap error kalau server/workflow-nya belum aktif. Ini ngajarin saya buat selalu cek dua sisi: sisi request (Postman) dan sisi server (n8n).
- Saya jadi lebih paham beda antara mode "test" dan mode "production" di webhook. Kalau masih coba-coba, testnya cuma sekali jalan. Kalau sudah production/publish, baru bisa dipanggil berkali-kali.

**Tantangan**
Paling sering ketemu error 404 "not registered" padahal URL-nya udah bener. Awalnya bingung, ternyata penyebabnya macam-macam — kadang salah slash, kadang workflow belum di-publish ulang, kadang harus restart server dulu. Dari sini saya belajar sabar dan teliti, coba dicek satu-satu penyebabnya sebelum nyerah.

**Kesan**
Role ini bikin saya ngerti bahwa jadi "penguji" itu penting banget, kita yang pertama kali tahu kalau ada yang tidak beres, sebelum orang lain (dosen atau pengguna) yang menemukannya.

---

### Role 2 — Aryadi (Protocol & Traffic Analyst)

**Apa yang dikerjakan**
Saya bertugas menangkap dan menganalisis traffic jaringan pakai Wireshark, buat membuktikan bahwa komunikasi antara Postman, backend API, dan n8n itu beneran terjadi lewat jaringan — bukan cuma tampilan di layar.

**Yang dipelajari**
- Data yang kita lihat di Postman itu sebenarnya cuma "hasil akhir". Di baliknya, ada banyak proses pengiriman paket data lewat jaringan yang bisa dilihat detailnya kalau kita pakai Wireshark.
- Saya belajar cara pakai filter di Wireshark (misalnya `tcp.port == 8088` atau `tcp.port == 5678`) supaya tidak kebanjiran data yang tidak relevan. Ini penting karena tanpa filter, traffic yang muncul bisa ratusan bahkan ribuan paket yang bikin bingung.
- Saya jadi lebih ngerti bedanya traffic waktu request berhasil sama waktu request gagal — ternyata isi paketnya beda, bukan cuma soal "muncul error" doang.

**Tantangan**
Kadang traffic yang mau dicapture susah dibedakan mana yang relevan sama use case kita, apalagi kalau ada aplikasi lain yang jalan bareng dan ikut kirim data lewat jaringan. Perlu latihan dan kesabaran buat nyaring data yang benar-benar dibutuhkan.

**Kesan**
Dari role ini saya jadi ngerti bahwa "bukti" itu tidak cukup cuma dari satu sumber. Kalau Postman bilang berhasil, itu baru satu sisi cerita.  Wireshark membuktikan hal yang sama dari sisi jaringan, jadi laporan kita jadi lebih kuat dan bisa dipercaya.

---

### Role 3 — Faza Qinthoro Putra Tsany (Integration/Workflow Engineer)

**Apa yang dikerjakan**
Saya bertugas membangun workflow di n8n — mulai dari terima request, cek validasi data (misalnya jumlah order harus lebih dari 0 dan nama customer tidak boleh kosong), sampai bikin response yang dikirim balik ke Postman, baik itu response sukses maupun gagal.

**Yang dipelajari**
- Membuat workflow itu mirip bikin peta jalan buat data — data masuk dari satu pintu (webhook), terus dicek, lalu dipecah jadi dua jalur: jalur "lolos" (sukses) dan jalur "tidak lolos" (gagal). Saya jadi ngerti cara berpikir yang lebih terstruktur soal alur data.
- Saya belajar bahwa workflow yang kelihatan sudah jadi di layar (canvas) itu belum tentu langsung bisa dipakai. Harus disimpan (save) dan diaktifkan (publish/active) dulu supaya beneran bisa dipanggil dari luar.
- Nambah endpoint baru itu ternyata bisa bikin endpoint lama ikut bermasalah kalau tidak di-restart atau di-publish ulang dengan benar. Ini ngajarin saya buat selalu cek ulang semua bagian, bukan cuma bagian yang baru saya ubah.

**Tantangan**
Yang paling ribet itu pas endpoint lama (yang sudah jalan duluan) tiba-tiba ikut error setelah saya nambahin fitur baru. Butuh waktu buat nyari tahu penyebabnya, sampai akhirnya ketahuan workflow-nya perlu di-restart dari server, bukan cuma dari tampilan browser.

**Kesan**
Role ini bikin saya lebih paham bahwa jadi "yang bangun sistem" itu tanggung jawabnya besar — satu perubahan kecil bisa berpengaruh ke banyak bagian lain. Jadi harus teliti dan selalu test ulang semuanya, bukan cuma bagian yang baru dikerjakan.

---

### Role 4 — Nabiel Azzacky (Documentation & Presenter Lead)

**Apa yang dikerjakan**
Saya bertugas mengumpulkan semua hasil kerja teman-teman (screenshot Postman, Wireshark, dan n8n), lalu menyusunnya jadi laporan yang rapi dan mudah dibaca, termasuk menyiapkan bahan buat presentasi.

**Yang dipelajari**
- Menyusun laporan itu bukan cuma copy-paste screenshot, tapi harus bisa menjelaskan **kenapa** setiap gambar itu penting dan **apa hubungannya** satu sama lain. Jadi saya harus benar-benar paham dulu apa yang dikerjakan Role 1, 2, dan 3 sebelum bisa menuliskannya dengan jelas.
- Saya belajar pentingnya konsistensi — misalnya penomoran gambar, istilah yang dipakai, sampai format tabel, semua harus sama dari awal sampai akhir supaya laporan enak dibaca.
- Saya jadi lebih ngerti alur project secara keseluruhan, dari request yang dikirim sampai jadi laporan akhir, karena tugas saya "menyambungkan" semua bagian jadi satu cerita utuh.

**Tantangan**
Kadang ada evidence dari teman-teman yang belum lengkap atau formatnya beda-beda, jadi saya harus komunikasi bolak-balik buat memastikan semua bagian sudah pas dan tidak ada yang kelewat sebelum laporan difinalisasi.

**Kesan**
Dari role ini saya belajar bahwa kerja tim itu hasil akhirnya tergantung dari komunikasi yang baik. Sebagus apapun kerja teknis Role 1, 2, dan 3, kalau tidak didokumentasikan dengan jelas, orang lain (termasuk dosen) akan susah paham apa yang sebenarnya sudah dikerjakan.

---

## 7. Referensi Endpoint (dari Tabel 1 Laporan)

| Endpoint/Action | Method | Fungsi | Skenario |
|---|---|---|---|
| `/api/health` | GET | Memastikan server backend aktif | Success |
| `/api/microservices/orders` | POST | Membuat order melalui REST API | Success/Failure |
| `/webhook/commproto-p10` | POST | Mengirim payload | Success/Failure |
| `/webhook/order-status/{orderId}` | GET | Mengecek status order | Found/Not Found |
