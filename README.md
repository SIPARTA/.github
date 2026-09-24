# SIPARTA: Sistem Informasi & Edukasi Keselamatan Bahaya Kimia

![SIPARTA Architecture](https://img.shields.io/badge/Architecture-Hybrid_Web2_Web3-blue?style=for-the-badge)
![Status](https://img.shields.io/badge/Status-Development-green?style=for-the-badge)

**SIPARTA** adalah platform peringatan dini dan mitigasi bahaya kimia berbasis **Real-Time IoT Detection**, **Artificial Intelligence (Gemini)**, dan **Blockchain (Polygon Amoy)**. Proyek ini memadukan sensor gas pintar di *edge device*, backend yang kokoh, penyimpanan terdesentralisasi, serta *frontend* modern untuk menghadirkan perlindungan keselamatan yang tidak bisa dimanipulasi secara sepihak.

---

## 1. Project Overview

Tujuan utama SIPARTA adalah memberikan peringatan detik itu juga saat terdeteksi gas berbahaya (seperti Amonia, Hidrogen, Propana, dsb) di laboratorium atau area industri, sekaligus mencatat riwayat insiden secara permanen di blockchain agar terjamin keaslian (auditabilitas) datanya. Ketika insiden bahaya terjadi, sistem akan mengirimkan data sensor ke Google Gemini AI untuk menghasilkan rekomendasi mitigasi dan evakuasi secara spesifik berdasarkan gas yang terdeteksi.

### Fitur Utama:
- **Real-Time IoT Detection**: Pembacaan sensor MICS-5524, TGS2600, MQ-2, MQ-135 secara periodik.
- **AI-Driven Mitigation**: Integrasi Gemini 2.5 Flash dengan profil AI konservatif untuk memandu evakuasi.
- **Immutable Audit Trail**: Pencatatan data sensor mentah dan ringkasan AI ke jaringan blockchain Polygon Amoy.
- **Premium Dashboard**: UI/UX berbasis Next.js dengan dukungan MetaMask wallet.

---

## 2. Architecture Overview

Arsitektur SIPARTA dibagi menjadi 5 modul terpisah yang terintegrasi secara asinkron.

### Daftar Komponen
1. **IoT Edge (`ai_models`)**: Skrip Python di Raspberry Pi yang membaca ADC (ADS1115), menjalankan model Neural Network lokal (TFLite), dan mengirim POST request.
2. **Backend Services (`web-backend`)**: Aplikasi FastAPI yang menerima payload dari IoT, berkomunikasi dengan Supabase, memanggil API Gemini, dan mengeksekusi Subprocess Node.js ke layer Blockchain.
3. **Blockchain Layer (`blockchain_services`)**: Skrip relayer Node.js (Hardhat/Thirdweb) yang membayarkan *gas fee* (meta-transactions) untuk menulis data ke *Smart Contract* di Polygon Amoy, serta mengunggah metadata gambar ke IPFS via Pinata.
4. **Frontend (`web-frontend`)**: Dasbor Next.js (React) modern yang menggunakan koneksi klien Supabase (baca histori) dan SDK Thirdweb (interaksi wallet pengguna).
5. **IoT UI Legacy (`iot-devices`)**: UI interaktif lokal berbasis PyQt5 yang berjalan pada layar perangkat Raspberry Pi.

### End-to-End Data Flow
1. **Deteksi Edge**: Raspberry Pi membaca lonjakan gas (contoh: Amonia tinggi) -> Prediksi TFLite menyatakan status `BAHAYA`.
2. **Transmisi**: RPi mengirim HTTP POST multipart (JSON + Gambar) ke endpoint FastAPI `/api/v1/incidents/report`.
3. **Proses Backend**: 
   - Backend memanggil **Gemini AI** dengan input sensor & gambar untuk mendapatkan *Mitigation Text*.
   - Backend men-spawn **Subprocess `relay.ts`**.
4. **Pencatatan Web3**: `relay.ts` mengunggah gambar ke **IPFS**, lalu menulis transaksi ke **Smart Contract** menggunakan *Relayer Wallet*.
5. **Database Web2**: Jika transaksi sukses, Backend menyimpan rekam jejak tersebut ke tabel `incidents` di **Supabase**.
6. **Frontend**: Pengguna (admin) membuka Dasbor web -> Data dirender secara *real-time*.

---

## 3. Prerequisites

Sebelum menjalankan SIPARTA, Anda wajib menyiapkan *environment* berikut di OS (Linux/MacOS/WSL):

- **Python 3.10+** (Untuk backend FastAPI & AI Models)
- **Node.js 20+** (Untuk Next.js frontend & Blockchain script)
- **npm / yarn** (Package manager Node.js)
- **Raspberry Pi OS / Linux** (Bila ingin menjalankan Edge Device secara fisik)
- **MetaMask Wallet Extension** (Untuk autentikasi di Frontend)
- **Akun Supabase** (Database Relasional & Storage)
- **Akun Google AI Studio** (Gemini API Key)
- **Akun Thirdweb & Pinata** (Web3 Infrastructure)

Verifikasi instalasi:
```bash
python --version
node -v
npm -v
```

---

## 4. Project Structure

```text
SIPARTA/
├── ai_models/            # Skrip IoT Edge & Inference (Raspberry Pi)
├── blockchain_services/  # Smart Contracts (Solidity), Relayer Node.js, Thirdweb
├── iot-devices/          # Aplikasi PyQt5 untuk UI lokal layar Raspberry Pi
├── web-backend/          # REST API (FastAPI) & Integrasi Supabase + Gemini
└── web-frontend/         # Dasbor Admin Premium (Next.js, Tailwind, Thirdweb)
```

---

## 5. Global Environment Configuration

Seluruh environment variable tidak boleh di-commit. Masing-masing direktori memiliki `.env.example`.

*Catatan: Nilai dari `.env` di `blockchain_services/` digunakan pula oleh `web-backend/` karena backend akan menjalankan script dari direktori tersebut.*

---

## 6. Installation & Setup End-to-End

Ikuti langkah-langkah di bawah secara berurutan.

### Tahap 1: Setup Blockchain (`blockchain_services`)
Lapisan ini wajib berjalan terlebih dahulu untuk menyiapkan alamat kontrak.

1. Pindah ke direktori: `cd blockchain_services`
2. Install Dependency: `npm install`
3. Salin Konfigurasi: `cp .env.example .env`
4. Isi `.env`:
   - `RELAYER_PRIVATE_KEY` (Wallet dengan saldo MATIC di Amoy testnet)
   - `POLYGON_AMOY_RPC_URL=https://polygon-amoy.drpc.org`
   - `PINATA_JWT`
5. Deploy Contract:
   ```bash
   npm run deploy:audit
   npm run deploy:certificate
   ```
6. *Catat* alamat kontrak (Contract Address) yang dihasilkan, dan masukkan kembali ke dalam file `.env` di variabel `SIPARTA_AUDIT_CONTRACT` & `SIPARTA_CERTIFICATE_CONTRACT`.

### Tahap 2: Setup Backend (`web-backend`)
Backend merupakan pusat pengolahan.

1. Pindah ke direktori: `cd ../web-backend`
2. Buat *Virtual Environment*: `python -m venv venv && source venv/bin/activate`
3. Install Dependency: `pip install -r requirements.txt`
4. Salin Konfigurasi: `cp .env.example .env`
5. Isi `.env`:
   - `SUPABASE_URL`, `SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`
   - `GEMINI_API_KEY`
   - `DEVICE_API_KEY` (Buat token acak untuk RPi, contoh: `secret123`)
6. Jalankan Server Development:
   ```bash
   uvicorn fastapi.main:app --reload
   ```
   *(Backend berjalan di `http://localhost:8000`)*

### Tahap 3: Setup Frontend (`web-frontend`)
Menyajikan antarmuka bagi Admin/Auditor.

1. Pindah ke direktori: `cd ../web-frontend`
2. Install Dependency: `npm install`
3. Salin Konfigurasi: `cp .env.example .env.local`
4. Isi `.env.local`:
   - `NEXT_PUBLIC_SUPABASE_URL`
   - `NEXT_PUBLIC_SUPABASE_ANON_KEY`
   - `NEXT_PUBLIC_THIRDWEB_CLIENT_ID`
   - `NEXT_PUBLIC_DISEASE_API_URL=http://localhost:8000` (atau URL backend di Render)
5. Jalankan Server Development:
   ```bash
   npm run dev
   ```
   *(Aplikasi berjalan di `http://localhost:3000`)*

### Tahap 4: Setup IoT Edge (`ai_models`)
Simulasi / eksekusi skrip RPi pengirim data.

1. Pindah ke direktori: `cd ../ai_models`
2. Install Dependency: `pip install -r requirements.txt` (Hanya di environment IoT/RPi)
3. Salin Konfigurasi: `cp .env.example .env`
4. Isi `.env`:
   - `SIPARTA_BACKEND_URL=http://localhost:8000` (Ganti dengan IP server backend jika berjalan di perangkat fisik terpisah)
   - `DEVICE_ID` (Gunakan UUID dari tabel Supabase)
   - `DEVICE_API_KEY=secret123`
5. Jalankan Perangkat:
   ```bash
   python main_rpi.py
   ```

---

## 7. Integration Setup & Data Flow

Semua layanan di atas telah terintegrasi berdasarkan konfigurasi port/endpoint standar:

- **IoT Edge -> Backend**: Menggunakan HTTP POST multipart/form-data ke `http://localhost:8000/api/v1/incidents/report` dengan header `X-API-Key`.
- **Backend -> Gemini API**: Dihubungkan menggunakan package `google-generativeai`. Di dalam `gemini_service.py`, backend akan menyisipkan gambar dan memparsing data sensor.
- **Backend -> Blockchain (Relayer)**: Melalui Python `subprocess.run()`, backend memanggil `npx ts-node src/relay.ts` di dalam direktori `blockchain_services`.
- **Backend -> Database**: Menyimpan objek utuh (termasuk Transaction Hash) ke tabel `incidents` di Supabase.

> [!WARNING]
> Saat merilis ke production, pastikan Anda menggunakan environment variables dari platform hosting (Render) untuk Backend dan Vercel untuk Frontend, alih-alih `localhost`.

---

## 8. Testing & Verification

Ikuti *checklist* berikut untuk memastikan instalasi Anda valid:

1. [x] **Frontend Load Test**: Buka `http://localhost:3000`. Coba login MetaMask. Jika gagal, periksa `NEXT_PUBLIC_THIRDWEB_CLIENT_ID`.
2. [x] **Backend Health Check**: Buka `http://localhost:8000/health`. Respons harus `"status": "ok"`.
3. [x] **IoT Trigger Simulation**: Jalankan `main_rpi.py` pada kondisi ada gas (atau manipulasi kode untuk `status: "BAHAYA"`).
4. [x] **End-to-End Test**:
   - Perhatikan terminal Backend: Apakah ia mencetak `Calling Gemini AI...` lalu `[WEB3] Tx Hash: 0x...`?
   - Cek tabel `incidents` di dashboard Supabase. Apakah *row* baru terbuat?
   - Cek `http://localhost:3000/monitoring`. Apakah kartu insiden baru muncul beserta peringatan merah *Disclaimer* AI?

---

## 9. Troubleshooting

### 1. Symptom: Relay Blockchain Gagal (`subprocess` error di Backend)
- **Penyebab**: Dependency Node.js di `blockchain_services` belum diinstal atau `RELAYER_PRIVATE_KEY` salah/kehabisan gas.
- **Solusi**: Jalankan `npm install` di dalam direktori `blockchain_services`. Pastikan RPC Polygon Amoy normal.

### 2. Symptom: Frontend gagal fetch histori insiden (Blank Screen)
- **Penyebab**: Supabase credentials salah di `.env.local` frontend.
- **Solusi**: Pastikan URL dan RLS (Row Level Security) tabel Supabase mengizinkan pembacaan.

### 3. Symptom: AI Analysis mengembalikan pesan statis "SISTEM AI GAGAL TERHUBUNG..."
- **Penyebab**: API Key Gemini tidak valid atau masalah jaringan.
- **Solusi**: Periksa `GEMINI_API_KEY` di backend. Fitur _fallback dinamis_ telah aktif secara _default_ untuk keamanan.

---

## 10. Security Notes

- **Private Keys**: Jangan *pernah* menaruh `RELAYER_PRIVATE_KEY` di frontend atau backend database. Ia hanya dikonsumsi oleh script Typescript di `.env` lokal blockchain.
- **API Abuse**: Backend dilindungi `DEVICE_API_KEY`. Pastikan menggunakan token yang kuat.
- **Prompt Injection Defense**: Controller backend (FastAPI) memvalidasi field status hanya menerima `AMAN`, `WASPADA`, `BAHAYA`. Jangan menghapus logika _sanitizer_ ini di `incidents.py`.

---

## 11. Production Checklist

Sebelum mempublikasikan sistem (Live Deployment), pastikan:
- [ ] Backend IP / Domain diatur di Vercel Frontend (`NEXT_PUBLIC_DISEASE_API_URL`).
- [ ] CORS Backend (`ALLOWED_ORIGINS` di `main.py`) disesuaikan ke URL Vercel.
- [ ] Hardhat deployment dipindah ke Polygon Mainnet (opsional jika siap live).
- [ ] RPi IoT Endpoint diubah dari `localhost` ke domain Render Backend.
- [ ] Row Level Security (RLS) Supabase diaktifkan agar publik tidak bisa menambah row ke tabel.
