# System Design & Implementation Plan: Interactive World Cup AI Photobooth System
**Document Version:** 1.0.0  
**Standard:** Enterprise SDLC Compliant  
**Author:** Principal System Architect  
**Date:** July 2026  

---

## 1. Executive Summary & Project Overview

### 1.1 Project Objective
Proyek ini bertujuan untuk membangun sistem *Interactive AI Photobooth* khusus untuk acara bertema World Cup. Sistem ini memanfaatkan kecerdasan buatan (*Computer Vision*) untuk memberikan pengalaman imersif bagi pengunjung. Pengguna dapat memilih tim/negara favorit, menggunakan filter wajah dinamis secara *real-time*, dan memicu pengambilan foto menggunakan gestur tangan (*hand gestures*). Hasil foto akan langsung diproses dan diunggah ke *platform web micro* agar dapat diunduh oleh pengguna secara instan via pemindaian *QR Code*.

### 1.2 Hardware Specification & Utilization
Sistem dirancang untuk berjalan secara optimal di atas infrastruktur perangkat keras *edge-computing* performa tinggi dengan spesifikasi berikut:
* **Compute Node:** Intel Core i7 Generasi 9, RAM 32 GB DDR4.
* **Dedicated GPU:** NVIDIA GeForce RTX 4080 16GB (Akan dieksploitasi penuh menggunakan CUDA core untuk akselerasi inferensi AI *real-time* dan komposisi grafis resolusi tinggi).
* **Display:** TV 50 Inci 4K UHD (Sebagai media *live feedback display* bagi pengguna).
* **Capture Device:** Webcam Resolusi Tinggi (Minimal 1080p @60fps dengan kemampuan kompensasi pencahayaan rendah).

---

## 2. System Architecture

Sistem ini mengadopsi arsitektur terdistribusi *hybrid* yang memisahkan antara *Edge Client Application* (pemrosesan video & AI lokal) dan *Centralized Web Service* (distribusi data dan manajemen berkas gambar).

### 2.1 High-Level Architecture Diagram
```
+------------------------------------------------------------------------+
|                          EDGE COMPUTING NODE                           |
|                                                                        |
|  +------------------+     +--------------------+     +--------------+  |
|  |  Webcam Feed     | --> | OpenCV Pipeline    | --> | TV 50" UHD   |  |
|  |  (1080p @60fps)  |     | (Frame Rendering)  |     | (Live View)  |  |
|  +------------------+     +---------+----------+     +--------------+  |
|                                     |                                  |
|                                     v                                  |
|                           +--------------------+                       |
|                           | MediaPipe AI Engine|                       |
|                           | (FaceMesh & Hands) |                       |
|                           | [Accelerated CUDA] |                       |
|                           +---------+----------+                       |
|                                     | (On Gesture Trigger)             |
|                                     v                                  |
|                           +--------------------+                       |
|                           | Photo Exporter     |                       |
|                           | (JPEG Compression) |                       |
|                           +---------+----------+                       |
+-------------------------------------|----------------------------------+
                                      |
                                      | HTTP POST (Multipart API)
                                      v
+------------------------------------------------------------------------+
|                         WEB APPLICATION SERVICE                        |
|                                                                        |
|  +------------------+     +--------------------+     +--------------+  |
|  |  Laravel API     | --> | File Storage       | --> | Client Phone |  |
|  |  Gateway        |     | (/public/storage)  |     | (QR Download)|  |
|  +--------+---------+     +--------------------+     +--------------+  |
|           |                                                            |
|           v                                                            |
|  +------------------+                                                  |
|  | Database Layer   |                                                  |
|  | (MySQL / SQLite) |                                                  |
|  +------------------+                                                  |
+------------------------------------------------------------------------+
```

### 2.2 Component Description
1.  **Edge AI Client (Python Engine):** Aplikasi desktop *standalone* yang bertanggung jawab penuh terhadap *aquisi* video dari kamera, kalkulasi koordinat spasial wajah dan tangan, rendering filter berbasis tim negara, penanganan state hitung mundur, pembuatan gambar akhir (*compositing*), penayangan QR Code, serta pengiriman data gambar ke cloud.
2.  **Web Management System (Laravel Stack):** Menyediakan RESTful API endpoints yang aman untuk menerima unggahan berkas foto dari Edge Client, melakukan manajemen data transaksi di dalam basis data, dan menyajikan halaman web unduhan yang *mobile-friendly* dan berbobot ringan (*low-latency*).

---

## 3. Component Design & Software Engineering

### 3.1 Edge Application State Machine (Python)
Aplikasi lokal bekerja berdasarkan siklus hidup *State Machine* berikut untuk menjaga konsistensi alur interaksi pengguna:

```
                  +-----------------------+
                  |      1. STANDBY       | <----------------------------+
                  +-----------+-----------+                              |
                              |                                          |
                              | (User Detection / Input)                 |
                              v                                          |
                  +-----------------------+                              |
                  |   2. TEAM SELECTION   |                              |
                  +-----------+-----------+                              |
                              |                                          |
                              | (Selection Confirmed)                    |
                              v                                          |
                  +-----------------------+                              |
                  |     3. LIVE FILTER    |                              |
                  +-----------+-----------+                              |
                              |                                          |
                              | (Gesture Trigger: Thumbs Up / High-Five) |
                              v                                          |
                  +-----------------------+                              |
                  |   4. COUNTDOWN & CAP  |                              |
                  +-----------+-----------+                              |
                              |                                          |
                              | (Image Saved)                            |
                              v                                          |
                  +-----------------------+                              |
                  | 5. PROCESSING & UPLOAD|                              |
                  +-----------+-----------+                              |
                              |                                          |
                              | (API Return Success URL)                 |
                              v                                          |
                  +-----------------------+                              |
                  |  6. DISPLAY QR CODE   | -----------------------------+
                  +-----------------------+   (Timeout 15 Seconds)
```

### 3.2 AI Core & Computer Vision Pipeline
* **Modul Filter Wajah:** Menggunakan `MediaPipe Face Mesh` untuk mendeteksi 468 titik tengara (*landmarks*) wajah. Koordinat 2D dari titik pipi, dahi, dan hidung diekstrak untuk menghitung matriks transformasi afin (*Affine Transformation*), memungkinkan filter bendera negara (`PNG Alpha Channel` yang didesain melalui Adobe Photoshop) terdistorsi dan menempel secara presisi mengikuti gerakan kepala pengguna.
* **Modul Deteksi Gestur:** Menggunakan `MediaPipe Hands`. Setiap frame diperiksa keberadaan struktur tangannya.
    * *Gestur High-Five:* Terpicu jika jarak vertikal ujung jari (`TIP`) lebih tinggi daripada buku jari (`PIP`) untuk semua lima jari secara bersamaan.
    * *Gestur Thumbs Up:* Terpicu jika jari jempol tegak lurus ke atas sementara keempat jari lainnya menekuk ke dalam telapak tangan (berdasarkan perbandingan jarak koordinat terhadap titik tengah telapak tangan/`WRIST`).

### 3.3 API Contract (Edge-to-Cloud Communication)
Komunikasi data dilakukan secara asinkronus menggunakan arsitektur REST API.

#### 3.3.1 Endpoint Upload Foto
* **HTTP Method:** `POST`
* **URL Endpoint:** `/api/v1/photobooth/upload`
* **Content-Type:** `multipart/form-data`
* **Payload Request:**
    ```json
    {
      "device_id": "EDGE-WC2026-01",
      "selected_team": "Indonesia",
      "photo_file": [BINARY_IMAGE_DATA_JPEG]
    }
    ```
* **Response Success (201 Created):**
    ```json
    {
      "status": "success",
      "message": "Photo uploaded successfully",
      "data": {
        "photo_id": "wb-78b1c9",
        "download_url": "https://yourdomain.com/d/wb-78b1c9",
        "expired_at": "2026-07-13 23:59:59"
      }
    }
    ```

---

## 4. Database Design (Backend Web)

Desain skema basis data dioptimalkan untuk performa baca/tulis yang cepat serta mendukung manajemen penghapusan otomatis data berkas lama.

### 4.1 Table Structure: `photos`
| Column Name | Data Type | Constraints | Description |
| :--- | :--- | :--- | :--- |
| `id` | bigint | Primary Key, Auto Increment | ID Internal Sistem |
| `uuid` | varchar(36) | Unique, Indexed | ID Unik untuk URL publik (UUIDv4) |
| `device_id` | varchar(50) | Not Null | Identitas mesin pengirim |
| `team_name` | varchar(50) | Not Null | Nama negara yang dipilih pengguna |
| `file_path` | varchar(255) | Not Null | Lokasi fisik penyimpanan berkas foto di server |
| `created_at` | timestamp | Nullable | Waktu pengambilan foto |
| `updated_at` | timestamp | Nullable | Waktu modifikasi records |

---

## 5. Security & Performance Optimization

1.  **Akselerasi GPU (CUDA):** Pemrosesan OpenCV dan inferensi MediaPipe dipaksa berjalan pada RTX 4080 melalui binding pustaka CUDA. Ini menjamin pemrosesan frame tetap konstan pada 60 FPS pada resolusi output tinggi tanpa membebani CPU Core.
2.  **Asynchronous API Call pada Edge:** Operasi I/O pengunggahan berkas foto ke web server dijalankan pada `Thread` terpisah (menggunakan modul `threading` Python). Langkah ini krusial agar UI aplikasi utama di TV tidak mengalami pembekuan (*freezing*) saat proses *upload* sedang berlangsung di latar belakang.
3.  **Data Retention Policy (Pembersihan Otomatis):** Untuk menghemat ruang penyimpanan server selama event World Cup berjalan, sebuah tugas terjadwal (*Cron Job*) melalui perintah `Schedule` Laravel diaktifkan untuk menghapus berkas foto fisik dan records di basis data yang umurnya telah melewati 24 jam dari waktu pembuatan.

---

## 6. Implementation Plan & SDLC Phases

Implementasi proyek dibagi ke dalam 5 fase terstruktur untuk memastikan manajemen risiko dan fungsionalitas sistem berjalan sempurna sebelum hari pelaksanaan acara.

```
Fase 1: Envir. Setup & Core AI --------> 2 Minggu
Fase 2: UI Engine & Edge Logic --------> 2 Minggu
Fase 3: Laravel API & Web Portal ------> 2 Minggu
Fase 4: Hardware Integration & UAT ----> 1 Minggu
Fase 5: Deployment & On-Site Run ------> 1 Minggu
```

### 6.1 Detail Breakdown Jadwal Kerja

#### Fase 1: Environment Setup & Core AI Engine (Minggu 1-2)
* Inisialisasi repositori Git dan pembagian branch development.
* Instalasi Python 3.10+, konfigurasi CUDA Toolkit 12.x dan cuDNN pada mesin i7/RTX4080.
* Pembuatan prototipe script Python untuk akses Webcam via OpenCV.
* Implementasi pipeline MediaPipe Face Mesh untuk penempatan mask filter dinamis.
* Pengembangan algoritma deteksi gestur *Thumbs Up* dan *High-Five* dengan nilai batas akurasi (*confidence threshold*) > 0.8.

#### Fase 2: UI Graphics Engine & Local Edge Logic (Minggu 3-4)
* Integrasi aset visual PNG bertema negara peserta World Cup ke dalam pipeline render OpenCV.
* Penyusunan modul logika State Machine (Standby, Selection, Live Filter, Countdown, Preview, QR Display).
* Desain layout antarmuka resolusi 4K untuk TV 50 inci agar tulisan dan tombol seleksi terbaca dengan jelas dari jarak 2-3 meter.
* Penerapan modul generator QR Code lokal pada Python yang mengkonversi teks string menjadi matriks gambar visual secara instan.

#### Fase 3: Laravel API Gateway & Web Portal (Minggu 5-6)
* Setup framework Laravel pada server produksi (atau VPS lokal dengan arsitektur jaringan publik).
* Pembuatan skema migrasi database dan konfigurasi *file storage symlink*.
* Implementasi API Controller untuk menghandle *Multipart Form-Data* unggahan gambar dari Python.
* Pembuatan modul enkripsi URL pendek/Slug (misal menggunakan Hashids atau UUID) untuk keamanan parameter unduhan.
* Desain halaman antarmuka web unduhan responsif (HTML/CSS murni/Tailwind) yang memiliki performa muat sangat instan di peramban seluler (Chrome/Safari).

#### Fase 4: Hardware Integration & User Acceptance Testing / UAT (Minggu 7)
* Perakitan fisik komponen: PC, Webcam, dan TV 50" dalam satu kesatuan booth *mock-up*.
* Pengujian ketahanan performa (*Stress Testing*) aplikasi Python selama minimal 6 jam nonstop.
* Pengukuran latensi proses pengunggahan foto dari lokal ke server publik pada kondisi bandwidth internet minimum (simulasi tethering seluler).
* Pelaksanaan skenario pengujian fungsionalitas (UAT):
    * *Test Case 01:* Deteksi wajah dan akurasi filter saat pengguna bergerak aktif.
    * *Test Case 02:* Kecepatan deteksi gestur tangan pada berbagai variasi intensitas cahaya ruangan.
    * *Test Case 03:* Keberhasilan pemindaian QR Code oleh berbagai jenis sistem operasi perangkat genggam (iOS dan Android).

#### Fase 5: Deployment, Deployment Final & On-Site Simulation (Minggu 8)
* Instalasi permanen sistem hardware pada area venue utama kegiatan World Cup.
* Kalibrasi akhir lensa kamera terhadap pencahayaan panggung atau booth khusus (*Ring Light alignment*).
* Aktivasi skrip monitoring log otomatis untuk mendeteksi *error run-time* secara instan.
* Serah terima sistem (*Handover*) kepada tim operasional lapangan dan penyediaan lembar panduan penanganan masalah cepat (*Quick Troubleshooting Sheet*).

---

## 7. Maintenance & Disaster Recovery Plan

Untuk mengantisipasi kendala teknis di lapangan saat event besar berlangsung, berikut prosedur penanganan darurat yang disiapkan dalam sistem:

* **Masalah 1: Koneksi Internet Terputus (Offline State)**
    * *Mitigasi:* Aplikasi Python akan mendeteksi kegagalan API secara otomatis. Gambar hasil foto tidak akan hilang melainkan disimpan sementara di dalam direktori penyimpanan lokal mesin (`/local_cache/`).
    * *Recovery:* Setelah internet kembali pulih, sistem Edge menyediakan fungsi *Force Sync* untuk mengunggah seluruh antrean foto lokal ke server Laravel secara berurutan.
* **Masalah 2: Webcam Tidak Terdeteksi / Diskoneksi Kabel**
    * *Mitigasi:* OpenCV menangkap eksepsi kegagalan pembacaan index kamera (`cv2.VideoCapture`).
    * *Recovery:* Layar TV secara otomatis akan beralih menampilkan petunjuk visual kepada kru lapangan untuk memeriksa koneksi kabel USB kamera, alih-alih menampilkan layar hitam eror (*crash screen*).