# Panduan Kelompok — UAS Communication Protocol
## Use Case 3: Task/Order Tracking Workflow (REST + n8n optional)

> **Asumsi kerja:** kalian melanjutkan base project praktikum P9–P14 (server lokal `http://localhost:8088` dan n8n di `http://localhost:5678`, sesuai `base_url` & `n8n_webhook_url` di Postman Collection yang sudah ada). Soal UAS eksplisit mengizinkan ini ("Project boleh melanjutkan praktikum P9-P14"). Kalau ternyata backend order-nya belum lengkap, panduan di bawah menandai bagian mana yang perlu ditambah.

---

## 0. Konsep Besar: State Machine Order

Sebelum bagi kerja, samakan dulu pemahaman alur order-nya. Ini yang akan didemokan di P16:

```
CREATED  --(validasi n8n)-->  CONFIRMED (success path)
                          \--> REJECTED  (failure path)
```

- **CREATED** → order masuk lewat `POST /api/microservices/orders`
- **Validasi** → payload order dikirim ke n8n webhook, di-cek (mis. `amount > 0` dan `customer` tidak kosong)
- **CONFIRMED** → branch sukses (`workflowBranch = TRUE_BRANCH_SUCCESS`)
- **REJECTED** → branch gagal (`workflowBranch = FALSE_BRANCH_ERROR`)

Pola branch ini **sudah ada di collection** (request "P10 n8n Webhook Trigger"), tinggal di-reskin jadi konteks order, bukan dibuat dari nol.

---

## 1. Pemetaan Endpoint Existing → Kebutuhan Use Case

| Request di Postman Collection | Method | Fungsi untuk Order Tracking | Skenario | PIC Utama |
|---|---|---|---|---|
| `00 Health Check` | GET | Sanity check sebelum demo | Success | Role 1 |
| `P11 Microservices Order Flow` | POST | **Create order** — entry point utama use case | Success (order valid) / Failure (payload rusak → 400) | Role 1 |
| `n8n_webhook_url` (di-rename jadi "Order Workflow Trigger") | POST | **Validasi & branching status order** | Success (`TRUE_BRANCH_SUCCESS`) / Failure (`FALSE_BRANCH_ERROR`) | Role 3 |
| `P12 Observability Capture Scenario` | POST | Menangkap log/execution trace dari proses order | Success | Role 3 & 2 |
| `P16 UAS Capture Demo` | POST | Snapshot evidence gabungan untuk laporan final | Success | Role 4 |
| `P10 Rate Limited API` | GET | *(opsional, nilai tambah reliability)* simulasi order service overload | 200 / 429 | Role 1 |
| `P10 Load Balanced API` | GET | *(opsional)* ilustrasi order service punya >1 backend/replica | Success | Role 2 |
| `P14 XML/Protobuf Convert` | POST | *(opsional bonus)* tunjukkan alternatif format tukar data order | Success | Role 1 jika waktu cukup |

Ini sudah memenuhi syarat minimum: **≥4 endpoint/step**, **≥2 success**, **≥2 failure** — tanpa perlu bikin sistem baru dari nol.

**Yang perlu dicek dulu (langkah 0, sebelum bagi kerja):** jalankan `P11 Microservices Order Flow` apa adanya, lihat response aslinya. Kalau belum ada field status (mis. `orderStatus`), itu jadi task kecil tambahan buat Role 1/3 untuk menambahkannya di kode backend/n8n — jangan asumsikan dulu formatnya sebelum dicek langsung.

---

## 2. Role 1 — API & Postman Tester

**Tanggung jawab inti (dari soal):** desain endpoint/action, buat request success & error, jalankan Postman Collection.

### Langkah kerja
1. Jalankan `00 Health Check` → pastikan server hidup (screenshot status 200).
2. Jalankan `P11 Microservices Order Flow` dengan payload valid → catat response asli (orderId, status, dsb).
3. Rancang **2 skenario sukses**:
   - Create order dengan payload lengkap & valid → expect `200/201`.
   - Cek status order (kalau backend belum punya endpoint GET status, tambahkan `GET /api/microservices/orders/{orderId}` sederhana, atau gunakan response dari webhook n8n sebagai "status check" — diskusikan dengan Role 3).
4. Rancang **2 skenario gagal**:
   - Create order dengan field wajib kosong/salah tipe (mis. `amount` string bukan number) → expect `400`.
   - Cek order dengan `orderId` yang tidak ada → expect `404` (atau pakai `P10 Rate Limited API` sebagai skenario gagal alternatif berbasis reliability, expect `429`).
5. Tambahkan `pm.test()` di setiap request baru — contoh pola sudah ada di request "P9 OpenAPI Spec" dan "P10 n8n Webhook Trigger", tinggal ditiru:
   ```js
   pm.test('Status code sesuai', function(){ pm.response.to.have.status(200); });
   pm.test('Response punya field orderStatus', function(){
     pm.expect(pm.response.json()).to.have.property('orderStatus');
   });
   ```
6. Rapikan collection: buat folder baru "Order Tracking Workflow" di dalam collection, isi 4 request inti di atas, request P10 lama biarkan sebagai folder terpisah "Reliability (opsional)".
7. Export ulang `postman/collection.json`.

### Evidence yang harus dikumpulkan
- [ ] Screenshot response sukses (status 200/201) + body
- [ ] Screenshot response gagal (status 400/404/429) + body
- [ ] Screenshot Test Results tab (semua `pm.test` hijau)
- [ ] File `postman/collection.json` final
- [ ] Raw request/response (untuk lampiran laporan bagian Implementasi)

---

## 3. Role 2 — Protocol & Traffic Analyst

**Tanggung jawab inti:** jelaskan protokol yang dipakai, observasi traffic (Wireshark/basic), baca header/payload relevan.

### Langkah kerja
1. Identifikasi protokol yang benar-benar dipakai di use case ini: **HTTP/REST** (client ↔ API order) dan **HTTP Webhook** (API/Postman → n8n). Use case 3 tidak wajib pakai MQTT/WebSocket, jadi fokus di dua ini saja — jangan dipaksakan pakai protokol lain.
2. Buka Wireshark, filter `tcp.port == 8088` (untuk trafik REST API) dan `tcp.port == 5678` (untuk trafik n8n webhook). Kalau Wireshark susah setup, alternatif "basic evidence" yang tetap diterima soal: tab **Postman Console** (Network tab) yang menunjukkan header lengkap request/response.
3. Saat Role 1 menjalankan request sukses & gagal, **capture bersamaan** di Wireshark/Console — jangan capture terpisah, supaya bisa dikorelasikan.
4. Untuk tiap screenshot, tulis **narasi analisis singkat** (wajib menurut soal — screenshot tanpa penjelasan tidak dihitung evidence):
   - Header apa yang relevan (`Content-Type: application/json`, correlation/request ID kalau ada)
   - Payload apa yang dikirim vs diterima
   - Kaitkan dengan status code yang dihasilkan
5. Tulis alasan pemilihan protokol untuk bagian laporan "Architecture Canvas & Protocol Selection": kenapa REST cocok untuk create/check order (stateless, gampang di-test), kenapa webhook/n8n cocok untuk orkestrasi validasi (logic branching dipisah dari API utama).
6. Pastikan minimal satu **error path** ikut ter-capture (syarat wajib data flow diagram).

### Evidence yang harus dikumpulkan
- [ ] Screenshot Wireshark/Console filter untuk request sukses
- [ ] Screenshot Wireshark/Console filter untuk request gagal
- [ ] Catatan analisis protokol (`docs/traffic-notes.md` atau langsung di laporan)
- [ ] Paragraf alasan pemilihan protokol untuk laporan bagian 3

---

## 4. Role 3 — Integration/Workflow Engineer (n8n)

**Tanggung jawab inti:** hubungkan komponen project; n8n boleh dipakai sebagai nilai tambah untuk webhook/workflow.

### Langkah kerja
1. Buka workflow n8n yang sudah ada dari praktikum (webhook path `/webhook/commproto-p10`). Rename/reskin jadi **"Order Tracking Workflow"** — logic branch-nya dipakai ulang, cuma konteksnya diganti jadi order:
   - **Webhook node**: terima payload `{ orderId, customer, amount }`
   - **IF/Switch node**: validasi bisnis, mis. `amount > 0 AND customer != ""`
   - **TRUE branch** → node "Build Success Response": set `orderStatus = "CONFIRMED"`, `workflowBranch = "TRUE_BRANCH_SUCCESS"`
   - **FALSE branch** → node "Build Error Response": set `orderStatus = "REJECTED"`, `workflowBranch = "FALSE_BRANCH_ERROR"`
   - **Respond to Webhook node**: kembalikan JSON gabungan (`workflowResult`, `workflowBranch`, `executedNode`, `backendStatusCode`, `orderStatus`)
2. Test **dua kali**: sekali dengan payload valid (harus lewat TRUE branch), sekali dengan payload invalid, mis. `amount: 0` (harus lewat FALSE branch). Ini otomatis memberi kalian 2 skenario sukses/gagal di level workflow — tambahan bukti selain yang di Postman.
3. Export workflow: n8n → menu **Download** → simpan sebagai `n8n/workflow.json`.
4. Screenshot **Execution History** untuk masing-masing branch (2 screenshot: execution sukses, execution gagal).
5. *(Nilai tambah, opsional)* Sambungkan output workflow ke `P12 Observability Capture Scenario` sebagai node HTTP Request tambahan di n8n — ini menunjukkan chaining REST → webhook → logging, memperkuat poin "microservices communication" di rubrik.
6. Bikin diagram flow sederhana (boxes + arrow + label protokol) buat dipakai Role 4 di architecture canvas & data flow diagram:
   ```
   Client (Postman) --HTTP/REST--> API Order (create)
                                       |
                                  HTTP Webhook
                                       v
                                 n8n Workflow (validasi & branch)
                                       |
                             +---------+---------+
                        TRUE_BRANCH            FALSE_BRANCH
                       (CONFIRMED)              (REJECTED)
                             |                       |
                             +---------+-------------+
                                       v
                              Response ke Client
   ```

### Evidence yang harus dikumpulkan
- [ ] `n8n/workflow.json` (hasil export)
- [ ] Screenshot execution history — branch sukses
- [ ] Screenshot execution history — branch gagal
- [ ] Diagram flow (untuk laporan & PPT)

---

## 5. Role 4 — Documentation & Presenter Lead

**Tanggung jawab inti:** README, laporan, PPT, folder evidence, narasi presentasi.

### Langkah kerja
1. **Susun struktur repo** persis sesuai soal (section 9):
   ```
   uas-commproto-kelompok-xx/
   ├─ README.md
   ├─ docs/ (laporan.pdf, laporan.docx, slides.pptx, architecture.png, data-flow.png)
   ├─ postman/collection.json
   ├─ n8n/workflow.json
   ├─ app/
   ├─ evidence/ (postman-success.png, postman-error.png, wireshark-capture.png, n8n-execution.png, logs.png)
   └─ reflection/kontribusi-anggota.md
   ```
2. **Kumpulkan evidence dari Role 1–3 secara berkala**, jangan nunggu H-1.
3. **README.md** wajib berisi: deskripsi use case, cara menjalankan (start server, start n8n, import Postman collection + set `base_url`/`n8n_webhook_url`), tabel role & kontribusi.
4. **Laporan (5–10 halaman, PDF+DOCX)** — ikuti struktur soal section 10:
   1. Cover (nama kelompok, NIM, kelas, dosen)
   2. Problem & use case (Order Tracking Workflow — kenapa relevan)
   3. Architecture canvas & alasan pilih protocol (dari Role 2 & 3)
   4. Data flow diagram + lokasi evidence
   5. Implementasi: endpoint, payload, response, status code (dari Role 1)
   6. Testing: success & failure scenario (dari Role 1)
   7. Reliability & observability (dari Role 2 & 3)
   8. Kontribusi anggota
   9. Limitation & improvement (jujur soal keterbatasan — soal eksplisit membolehkan ini)
   10. Lampiran screenshot + link repo
5. **PPT** — 7 slide sesuai soal section 11 (problem&tim / architecture / data flow&demo / evidence sukses / evidence gagal&reliability / observability / kontribusi&limitation&QA).
6. **Susun naskah presentasi** sesuai timing soal section 12 (total 7 menit + 3 menit QA), pastikan tiap anggota dapat bagian bicara:
   - 0–1 menit: problem & use case
   - 1–2 menit: architecture & protocol selection
   - 2–4 menit: demo success path (live)
   - 4–6 menit: demo failure/reliability
   - 6–7 menit: observability, limitation, closing
7. Tulis `reflection/kontribusi-anggota.md`.
8. **Cek ulang section 14 soal** (larangan academic): tidak ada token/password/API key di screenshot manapun.
9. Final check pakai checklist section 15 soal, lalu bikin ZIP/push ke GitHub, upload individual ke edlink sesi 8.

### Evidence yang harus dikumpulkan
- [ ] README.md
- [ ] laporan.pdf + laporan.docx
- [ ] slides.pptx
- [ ] reflection/kontribusi-anggota.md
- [ ] Repo/ZIP final

---

## 6. Timeline 1 Minggu

| Hari | Fokus |
|---|---|
| H1 | Semua: samakan konsep state machine order. Role 1 cek response asli endpoint existing. Role 3 cek akses n8n. |
| H2–H3 | Role 1: bangun/test 4 skenario (2 sukses, 2 gagal). Role 3: bangun & test n8n Order Workflow (2 branch). Role 2: mulai capture traffic paralel. |
| H4 | **Integration test bareng** (Role 1+2+3): jalankan full flow, capture evidence gabungan. Role 4: mulai README + architecture canvas. |
| H5 | Role 2 selesaikan traffic notes. Role 4 draft laporan & PPT skeleton dari evidence yang masuk. |
| H6 | Dry-run presentasi penuh, cek gap evidence, finalisasi dokumen. Role 4 compile ZIP/repo. |
| H7 | Final check checklist, upload edlink, presentasi P15/P16. |

---

## 7. Cek Silang ke Rubrik (100 Poin) — Biar Effort Terarah

| Area Rubrik | Bobot | Role Penanggung Jawab Utama |
|---|---|---|
| Desain arsitektur & protocol selection | 20 | Role 2 & 3 (isi), Role 4 (visual diagram) |
| Implementasi & demo end-to-end | 20 | Role 1 & Role 3 |
| Testing success/failure & reliability | 20 | Role 1 |
| Observability & traffic evidence | 15 | Role 2 & Role 3 |
| Dokumentasi GitHub/PDF/PPT | 10 | Role 4 |
| Kontribusi individu & role evidence | 10 | Semua (Role 4 kompilasi) |
| Presentasi kelompok & Q&A | 5 | Semua |

**Prioritas kalau waktu mepet:** fokus dulu ke 60 poin pertama (arsitektur, implementasi, testing) — itu semua ada di tangan Role 1 & 3. Observability (Role 2&3) jangan ditinggal karena bobotnya 15, tapi dokumentasi (10) & kontribusi (10) paling gampang dikejar terakhir asal evidence dari role lain sudah rapi.

---

## 8. Checklist Final (dari soal, section 15)

- [ ] Identitas kelompok & role anggota lengkap
- [ ] Use case & scope jelas (Order Tracking Workflow)
- [ ] Architecture canvas ada
- [ ] Data flow diagram ada (protokol dilabel di tiap panah, ada error path)
- [ ] Minimal 4 endpoint/action/workflow step
- [ ] Minimal 2 success scenario
- [ ] Minimal 2 failure/error scenario
- [ ] Postman Collection bisa di-import
- [ ] Wireshark/traffic evidence ada
- [ ] Observability/log/execution evidence ada
- [ ] README run command jelas
- [ ] PDF & DOCX laporan tersedia
- [ ] PPT presentasi tersedia
- [ ] GitHub/ZIP bisa dibuka
- [ ] Tidak ada secret/token/password di evidence manapun
