# Laporan Praktikum Modul 1 (Pertemuan 4): Kontainerisasi Docker & Orkestrasi Multi-Tier (Lab 1 & Lab 2)

- **Nama Mahasiswa**: Salsabila Nur Shafa
- **NIM**: 42430040
- **Kelas / Prodi**: Teknologi Informasi
- **Tautan Branch / Pull Request GitHub**: https://github.com/salsa-rgb/docker-praktikum1-42430040/tree/lab1-2/42430040-salsa
- **Nama Rekan Peer-Reviewer**: 42430042 - I Made Dedy Wanditya 

---

## 1. Tabel Komparasi Eksperimen Individu (Tahap 1 vs. Tahap 2 vs. Tahap 3)

| Parameter Pembanding | Tahap 1: Tanpa Docker (Native di Host OS) | Tahap 2: Docker Single-Stage (`Dockerfile.single`) | Tahap 3: Docker Multi-Stage (`Dockerfile` Lab 1) |
| :--- | :--- | :--- | :--- |
| **Kebutuhan Instalasi Go di Host** | Wajib instal Go SDK di laptop | Tidak wajib (cukup Docker, Go ada di dalam image) | Tidak wajib (cukup Docker, kompilasi terjadi di stage builder) |
| **Ukuran Image (`docker images`)** | Tidak menghasilkan image | `1.32 GB` (disk usage) / `322 MB` (content size) | `26.5 MB` (disk usage) / `7.71 MB` (content size), di bawah target 100 MB |
| **Output `container_or_host_id` & NIM** | Hostname: `DESKTOP-L0SGMSA` \| NIM: `42430040` | Container ID: `b86fe72e1b8f` & `fcb9bb04d80f` \| NIM: `42430040` | Container ID: `01cf7381cd37` \| NIM: `42430040` |
| **User Eksekusi Proses (`id`)** | User laptop saat ini | `uid=0(root)` | `uid=10001(appuser)` |
| **Menjalankan 2 Instansi Sekaligus** | Gagal jika port sama (bind error) | Berhasil (`-p 8081:8080` & `-p 8082:8080`) | Berhasil (-p 8081:8080 & -p 8082:8080) |

---

## 2. Bukti Screenshot & Penjelasan Analisis Setiap Tahapan

### A. Tahap 1 — Eksekusi Aplikasi Mandiri Tanpa Docker (Baseline)

![Bukti Screenshot Tahap 1](docs/screenshots/sc01-tahap1-tanpa-docker.png)

**Penjelasan Analisis Tahap 1:**

Pada tahap ini aplikasi Backend Go dijalankan langsung di sistem operasi laptop (Windows) tanpa Docker, memakai `go run main.go`. Endpoint `/healthz` menampilkan NIM 42430040, nama Salsa, dan `container_or_host_id` berisi `DESKTOP-L0SGMSA`. Nilai itu adalah nama komputer (hostname) laptop saya, karena fungsi `os.Hostname()` membaca nama mesin tempat proses berjalan, dan tanpa container, mesin itu adalah laptop saya sendiri.

Ketika instansi kedua dijalankan di terminal lain, aplikasi langsung berhenti dengan pesan `bind: Only one usage of each socket address (protocol/network address/port) is normally permitted` (padanan di Windows untuk `bind: address already in use`) dan `exit status 1`. Penyebabnya, satu port (8080) pada satu host hanya bisa dipakai oleh satu proses. Karena kedua aplikasi berbagi jaringan dan port yang sama di host, instansi kedua bentrok dengan instansi pertama. Tanpa isolasi, aplikasi saling berebut sumber daya host. Selain itu, aplikasi hanya bisa berjalan jika Go SDK terpasang di laptop.

### B. Tahap 2 — Kontainer Tunggal dengan Single-Stage Dockerfile

![Bukti Screenshot Tahap 2](docs/screenshots/sc02-tahap2-single-stage.png)

**Penjelasan Analisis Tahap 2:**

Pada tahap ini backend Go dibungkus ke dalam image `backend-api:42430040-single` memakai `Dockerfile.single` (base image `golang:1.22-bookworm`, `COPY . .`, lalu `go build`). Dari image yang sama saya menjalankan dua container, `kontainer-pertama` (port 8081) dan `kontainer-kedua` (port 8082).

**Perubahan `container_or_host_id`.** Pada Tahap 1 nilainya adalah hostname laptop (`DESKTOP-L0SGMSA`). Sekarang nilainya berubah menjadi ID container, yaitu `b86fe72e1b8f` untuk container pertama dan `fcb9bb04d80f` untuk container kedua. Hal ini terjadi karena aplikasi berjalan di dalam container, dan `os.Hostname()` di sana mengembalikan hostname container yang secara bawaan berisi ID container. Karena kedua ID berbeda, terbukti dua instansi tersebut berjalan sebagai dua lingkungan terpisah.

**Mengapa dua aplikasi bisa berjalan bersamaan tanpa bentrok port.** Setiap container memiliki lingkungan jaringan sendiri, sehingga kedua aplikasi sama-sama mendengarkan port 8080 di dalam container masing-masing tanpa saling mengganggu. Yang harus berbeda hanya port di sisi laptop (host). Opsi `-p 8081:8080` memetakan port 8081 di laptop ke port 8080 di dalam `kontainer-pertama`, sedangkan `-p 8082:8080` memetakan port 8082 ke port 8080 di dalam `kontainer-kedua`. Berbeda dengan Tahap 1, saat dua proses langsung berbagi port 8080 milik host dan instansi kedua gagal.

**Mengapa ukuran image single-stage membengkak.** Image ini berukuran 1.32 GB (disk usage) atau 322 MB (content size terkompresi), jauh di atas target 100 MB. Penyebabnya, base image `golang:1.22-bookworm` membawa seluruh Go SDK, compiler, peralatan build, dan paket sistem operasi Debian, ditambah kode sumber serta hasil unduhan dependensi yang tertinggal di layer image. Padahal untuk menjalankan aplikasi hanya dibutuhkan satu file hasil kompilasi, `server-app`. Selain itu, proses di dalam container berjalan sebagai `uid=0(root)`, yang kurang aman. Dua masalah ini yang akan diperbaiki pada Tahap 3 dengan multi-stage build.

### C. Tahap 3 (Lab 1 RPS) — Optimasi Dockerfile Multi-Stage Build (< 100 MB)

![Bukti Screenshot Tahap 3](docs/screenshots/sc03-tahap3-multistage.png)

**Penjelasan Analisis Tahap 3:**

Pada tahap ini saya membuat `backend/Dockerfile` dengan teknik multi-stage build. Image hasilnya, `backend-api:42430040-multistage`, berukuran 26.5 MB (disk usage) atau 7.71 MB (content size), sedangkan image single-stage dari Tahap 2 berukuran 1.32 GB (disk usage) atau 322 MB (content size). Artinya ukuran image turun sekitar 98% (dari 1.32 GB menjadi 26.5 MB berdasarkan disk usage), dan jauh di bawah batas 100 MB.

**Fungsi `COPY --from=builder`.** Dockerfile ini punya dua stage. Stage `builder` memakai `golang:1.22-alpine` untuk mengunduh dependensi dan mengompilasi `main.go` menjadi satu file biner `app-server`. Stage `production` memakai `alpine:3.20`, sistem operasi yang sangat kecil, dan instruksi `COPY --from=builder` hanya menyalin file biner itu dari stage pertama. Seluruh Go SDK, compiler, kode sumber, dan cache build tertinggal di stage `builder` dan dibuang, sehingga tidak masuk ke image akhir. Ukuran juga dibantu oleh `CGO_ENABLED=0`, yang menghasilkan biner mandiri, dan `-ldflags="-s -w"`, yang memangkas simbol debug. Selain itu, `go.mod` dan `go.sum` disalin lebih dulu sebelum `main.go` agar layer unduhan library tersimpan di cache: jika hanya `main.go` yang berubah, Docker tidak perlu mengunduh ulang library.

**Alasan memakai `USER appuser`.** Pada Tahap 2 proses berjalan sebagai `uid=0(root)`. Di Tahap 3 saya membuat user biasa `appuser` (uid 10001) dan menjalankan aplikasi dengannya, terbukti dari hasil `docker exec uji-multistage id` yang menampilkan `uid=10001(appuser)`. Dengan begitu, jika aplikasi berhasil diserang, penyerang hanya memperoleh hak akses terbatas, bukan root di dalam container. Ini penerapan prinsip hak akses minimum (least privilege) dan praktik keamanan standar untuk container produksi.

### D. Tahap 4 (Lab 2 RPS) — Orkestrasi Multi-Tier & Persistensi Data dengan Docker Compose

![Bukti Screenshot Tahap 4](docs/screenshots/sc04-tahap4-compose-persistensi.png)

**Penjelasan Analisis Tahap 4:**

Pada tahap ini tiga container dijalankan sekaligus dengan Docker Compose: `mk40-db` (PostgreSQL 16), `mk40-backend` (API Go hasil multi-stage build Tahap 3, dengan `STORAGE_MODE=postgres`), dan `mk40-frontend` (Nginx). Ketiganya berada di satu custom bridge network, `mk40_bridge_network`, sehingga bisa saling memanggil memakai nama layanan.
 
**Alur komunikasi.** Browser mengakses `http://localhost:8080`, yang dipetakan ke port 80 container `frontend` (satu-satunya yang memakai `ports:`). Nginx menyajikan halaman `index.html`, dan untuk permintaan `/api/` serta `/healthz` ia bertindak sebagai reverse proxy lewat `proxy_pass http://backend:8080`. Backend lalu menghubungi database dengan `DB_HOST=db`, yaitu nama layanan yang diterjemahkan otomatis oleh DNS internal Docker menjadi alamat container database. Pada hasil `docker compose ps`, `mk40-db` hanya menampilkan `5432/tcp` dan `mk40-backend` hanya `8080/tcp`, tanpa pemetaan `0.0.0.0`, sehingga port 5432 tidak terbuka ke publik dan database hanya bisa diakses dari dalam jaringan internal.
 
**Bukti persistensi data.** Saya menambahkan satu data lewat formulir, yaitu `Billing-Service-42430040`. Kemudian saya menjalankan `docker compose down` (container dan network terhapus) lalu `docker compose up -d`. Container ID backend yang tampil di halaman berubah dari `282e4c1c855c` menjadi `6f86dad17de5`, yang membuktikan container benar-benar dibuat baru. Namun data `Billing-Service-42430040` beserta dua data awal dari `init.sql` tetap tampil di tabel. Hal ini terjadi karena data PostgreSQL disimpan di named volume `pgdata` (bernama `mk40_pgdata_volume`) yang dipasang ke `/var/lib/postgresql/data`. Volume dikelola Docker di luar siklus hidup container, sehingga `docker compose down` (tanpa `-v`) tidak menghapusnya, dan container database yang baru memakai kembali data yang sama.

### E. Tahap 5 — Skenario Troubleshooting & Analisis Error

![Bukti Screenshot Tahap 5](docs/screenshots/sc05-tahap5-troubleshooting.png)

**Penjelasan Analisis Tahap 5:**
Pada eksperimen ini saya mengubah `DB_HOST` di service `backend` dari `"db"` menjadi `"localhost"`, lalu menjalankan `docker compose up -d` sehingga container backend dibuat ulang. Hasil `curl http://localhost:8080/healthz` menunjukkan `"status":"unhealthy"` dengan pesan `dial tcp [::1]:5432: connect: connection refused`, dan halaman web menampilkan `UNHEALTHY` dengan Container ID `undefined` serta tabel yang kosong. Setelah `DB_HOST` saya kembalikan ke `"db"` dan `docker compose up -d` dijalankan lagi, `/healthz` kembali `healthy` (Container ID backend `16fccae3d5ad`) dan data di tabel tampil kembali.
 
**Mengapa `localhost` bukan container `db`.** Setiap container memiliki lingkungan jaringan sendiri. Di dalam container `backend`, `localhost` (alamat loopback, tertulis `::1` pada log di atas, padanan IPv6 dari `127.0.0.1`) merujuk ke container backend itu sendiri, bukan ke container lain maupun ke laptop. Di dalam container backend tidak ada PostgreSQL yang mendengarkan port 5432, sehingga koneksi ditolak (`connection refused`). Container database hanya bisa dijangkau lewat nama layanannya, `db`, yang diterjemahkan oleh DNS internal Docker di custom network menjadi alamat container database.
 
**Bahaya `docker compose down -v`.** Flag `-v` membuat Docker menghapus named volume yang dideklarasikan di Compose, dalam hal ini `mk40_pgdata_volume` tempat seluruh data PostgreSQL disimpan. Tanpa `-v`, `docker compose down` hanya menghapus container dan network sehingga data tetap aman (seperti terbukti di Tahap 4). Dengan `-v`, data hilang permanen, dan saat `docker compose up` berikutnya database dibuat kosong dan hanya berisi data awal dari `init.sql`. Karena itu perintah ini berbahaya dijalankan pada lingkungan yang menyimpan data penting.

### F. Tantangan Mandiri — Dual Bridge Network, `.env`, dan Healthcheck Backend

![Bukti Screenshot Tantangan Mandiri](docs/screenshots/sc06-tantangan-mandiri.png)

**Penjelasan Analisis Tantangan Mandiri:**

Pada tantangan ini saya mengubah `docker-compose.yml` dengan tiga perbaikan: memindahkan kredensial ke file `.env`, memisahkan jaringan menjadi dua, dan menambahkan healthcheck pada backend.
 
**Dual Bridge Network.** Jaringan tunggal sebelumnya saya ganti dengan dua custom bridge network: `frontend-net` (`mk40_frontend_net`) dan `backend-net` (`mk40_backend_net`). Container `frontend` hanya terhubung ke `frontend-net`, container `db` hanya terhubung ke `backend-net`, sedangkan `backend` terhubung ke keduanya sehingga menjadi perantara. Akibatnya, perintah `docker exec mk40-frontend ping -c 2 db` gagal dengan `ping: bad address 'db'`: DNS internal Docker hanya menerjemahkan nama layanan yang berada di jaringan yang sama, dan frontend tidak berbagi jaringan dengan database. Meski begitu, aplikasi web di `http://localhost:8080` tetap berjalan normal (status `HEALTHY`, Container ID backend `efaaf7327539`, data tetap tampil), karena frontend hanya berkomunikasi dengan backend lewat `frontend-net`, lalu backend yang menghubungi database lewat `backend-net`. Pada `docker compose ps`, database hanya menampilkan `5432/tcp` tanpa pemetaan ke host, dan `docker compose port db 5432` tidak menghasilkan alamat publik (hanya pesan `invalid IP:0`), jadi port database tidak terbuka. Dengan pemisahan ini, jika frontend yang menghadap publik berhasil diretas, penyerang tetap tidak bisa menjangkau database secara langsung.
 
**Perlindungan kredensial dengan `.env`.** Nilai sensitif (`POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `STUDENT_NIM`, `STUDENT_NAME`) saya pindahkan dari `docker-compose.yml` ke file `.env`, dan di Compose dipanggil dengan substitusi `${NAMA_VARIABEL}`. File `.env` yang berisi nilai asli saya masukkan ke `.gitignore`, sehingga tidak ikut ter-commit dan tidak bocor ke GitHub. Yang di-commit hanya `.env.example`, berisi nama variabel dengan nilai contoh tanpa password asli, sebagai petunjuk bagi orang lain untuk membuat `.env` mereka sendiri.
 
**Healthcheck backend.** Pada service `backend` saya menambahkan healthcheck `wget --spider -q http://localhost:8080/healthz`, dan pada `frontend` saya mengatur `depends_on` dengan `condition: service_healthy`. Dengan begitu frontend baru dijalankan setelah backend benar-benar sehat (yang berarti koneksi ke database juga berhasil), sehingga pengguna tidak melihat halaman yang gagal memuat data.

---

## 3. Bukti Audit Silang (Peer-Review) di GitHub

- **Tautan Pull Request (PR)**: [Isi URL PR Anda]
- **Ringkasan Catatan Peer-Reviewer**: [Tuliskan hasil verifikasi dari rekan yang memeriksa PR Anda]