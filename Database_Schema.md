# Skema Database: Smart QR Menu & Self Ordering System

Untuk merealisasikan "Golden Flow" (Alur Krusial) pemesanan yang mulus, kita bisa memulai dengan struktur database yang sederhana namun fungsional. 

Berikut adalah rancangan tabel inti yang dibutuhkan untuk sistem Andrezo Cafe:

## 1. Tabel `categories`
Tabel ini digunakan untuk mengelompokkan menu agar pelanggan mudah mencari (misal: "Signature Coffee", "Snacks", "Main Course").

| Kolom | Tipe Data | Keterangan |
| :--- | :--- | :--- |
| `id` | INT (PK) | ID unik kategori |
| `name` | VARCHAR | Nama kategori (contoh: "Coffee") |
| `image_url` | VARCHAR | (Opsional) Ikon/gambar kategori |
| `sort_order` | INT | Urutan tampil di aplikasi |

## 2. Tabel `products`
Menyimpan informasi detail tentang menu makanan atau minuman. Di sinilah Anda akan memasukkan 3-5 menu andalan hasil riset Anda.

| Kolom | Tipe Data | Keterangan |
| :--- | :--- | :--- |
| `id` | INT (PK) | ID unik produk |
| `category_id` | INT (FK) | Relasi ke tabel `categories` |
| `name` | VARCHAR | Nama menu (contoh: "Kopi Susu Andrezo") |
| `description` | TEXT | Deskripsi menarik tentang menu |
| `price` | DECIMAL | Harga produk |
| `image_url` | VARCHAR | Link ke foto produk berkualitas tinggi |
| `is_available` | BOOLEAN | Status ketersediaan (Bisa dipesan atau Habis) |

## 3. Tabel `product_modifiers` (Opsional tapi Direkomendasikan)
Untuk memberikan pengalaman yang lebih nyata, tambahkan opsi kustomisasi (Add-ons). Ini membuat sistem terasa canggih.

| Kolom | Tipe Data | Keterangan |
| :--- | :--- | :--- |
| `id` | INT (PK) | ID unik kustomisasi |
| `product_id` | INT (FK) | Relasi ke tabel `products` |
| `name` | VARCHAR | Nama opsi (contoh: "Less Sugar", "Extra Espresso") |
| `additional_price`| DECIMAL | Harga tambahan (0 jika gratis) |

## 4. Tabel `orders`
Menyimpan data pesanan secara keseluruhan setelah pelanggan melakukan *checkout*.

| Kolom | Tipe Data | Keterangan |
| :--- | :--- | :--- |
| `id` | INT (PK) | ID unik pesanan |
| `table_number` | VARCHAR | Nomor meja (didapat dari scan QR code) |
| `customer_name` | VARCHAR | Nama pelanggan (opsional, untuk dipanggil) |
| `total_amount` | DECIMAL | Total harga seluruh pesanan |
| `status` | VARCHAR | Status: "Pending", "Diproses", "Selesai" |
| `created_at` | TIMESTAMP | Waktu pesanan dibuat |

## 5. Tabel `order_items`
Karena satu pesanan (`orders`) bisa berisi banyak produk, tabel ini mencatat detail per item yang dipesan.

| Kolom | Tipe Data | Keterangan |
| :--- | :--- | :--- |
| `id` | INT (PK) | ID unik detail pesanan |
| `order_id` | INT (FK) | Relasi ke tabel `orders` |
| `product_id` | INT (FK) | Relasi ke produk yang dipesan |
| `quantity` | INT | Jumlah pesanan untuk produk ini |
| `subtotal` | DECIMAL | (Harga Produk + Modifier) * Quantity |
| `notes` | TEXT | Catatan khusus ("jangan pakai bawang") |

---

### Langkah Selanjutnya:
1. Kumpulkan data dari GoFood atau Google Maps untuk mengisi tabel `categories` dan `products`.
2. Setelah data terkumpul, kita bisa mulai membuat mock-up UI (User Interface) atau memilih *framework* apa yang ingin Anda gunakan untuk membangun sistem ini (apakah berbasis Web dengan React/Vue/Laravel, atau lainnya).
