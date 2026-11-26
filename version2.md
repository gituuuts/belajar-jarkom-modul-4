# Modul 4 - IP Addressing, Komunikasi Data & Routing Antar Subnet
 
Panduan lengkap: **apa yang dicari → apa yang di-run → apa yang dibuktikan**, dari Bagian A sampai D + Tugas Mandiri.
 
> **Cara pakai file ini**
> 1. Ganti semua `[PFX]` dengan IP Prefix kamu (dua oktet pertama, contoh `10.10` atau `172.16`) pakai *Find & Replace* di editor. Jadi `[PFX].20.129` menjadi `10.10.20.129`.
> 2. Kerjakan berurutan: A (kertas) → B → C → D. Tiap bagian memakai hasil bagian sebelumnya.
> 3. Setiap kali ada tulisan **PREDIKSI**, tulis prediksimu di laporan/kertas **sebelum** menjalankan perintah. Ini wajib menurut modul.
> 4. Setiap kali ada tulisan **SCREENSHOT**, itu bukti yang diminta tabel observasi.
 
---
 
## 0. Gambaran Besar: Sebenarnya Modul Ini Mau Ngapain?
 
Satu kalimat: **membuat tiga (lalu empat) ruangan yang masing-masing punya subnet sendiri bisa saling ping, lalu membandingkan cara manual (static routing) dengan cara otomatis (OSPF).**
 
Alurnya seperti cerita:
 
| Bagian | Pertanyaan yang dijawab | Yang dibuktikan |
|---|---|---|
| **A** (kertas) | Satu blok `/24` dibagi jadi subnet-subnet berbeda ukuran seperti apa? Alamat tiap interface apa? Route apa yang dibutuhkan tiap router? Seberapa repot kalau jaringan membesar? | VLSM, rencana routing table, dan static routing **tidak scalable** (rumus `n(n-1)`) |
| **B** | Bagaimana router meneruskan paket antar subnet? | (1) Router hanya kenal *connected route* sampai kamu pasang route. (2) **Return route sama pentingnya dengan forward route**. (3) TTL turun 1 tiap hop dan checksum header berubah. (4) Summarization, Longest Prefix Match, dan routing loop |
| **C** | Apa yang terjadi kalau ada link cadangan lalu link utama putus? Apakah OSPF lebih baik? | Static route **tidak tahu link mati** (traffic hilang walau ada jalur lain). OSPF mendeteksi dan pindah jalur otomatis |
| **D** | Bagaimana kalau backbone dipakai bersama (broadcast) dan ada router baru? | DR/BDR, ECMP (load balancing 2 jalur), dan **menambah Ruang Baca cukup konfigurasi R4 saja**, R1-R3 tidak disentuh |
 
Inti materi yang diuji (hafalkan ini):
 
* **Router hanya meneruskan paket kalau punya route ke tujuan.** Paket pulang (reply) juga butuh route. Tanpa itu ping gagal walau paket pergi sampai.
* **Connected route** muncul otomatis saat interface diberi IP. **Static route** harus dipasang manual per tujuan.
* **Longest Prefix Match (LPM)**: kalau beberapa route cocok, yang prefix-nya paling panjang menang.
* **Administrative Distance (AD)**: kalau prefix sama dari sumber berbeda, yang AD lebih kecil menang (connected 0, static 1, OSPF 110, RIP 120). Di static, `distance=10` dipakai untuk membuat route cadangan.
* **TTL** berkurang 1 tiap router; kalau 0, router membuang paket dan mengirim ICMP *Time Exceeded*. Karena TTL berubah, **Header Checksum dihitung ulang** di tiap hop.
* **OSPF** = router saling bertukar info link, hitung jalur terbaik sendiri (cost), dan pindah otomatis kalau link mati.
---
 
## 1. Persiapan (Pra-requisite)
 
* GNS3 VM dan GNS3 GUI sudah terpasang.
* Appliance **MikroTik CHR** dan **netics-pc** sudah di-import. Untuk Laterano gunakan **netics-pc-desktop**.
* Tiap node CHR harus punya **5 network adapter** (ether1 sampai ether5), karena fungsi port dibakukan:
| Port | Fungsi | Dipakai di |
|---|---|---|
| ether1 | LAN ruang (alamat = default gateway host) | B, C, D |
| ether2 | Link p2p ke neighbor pertama | B, C |
| ether3 | Link p2p ke neighbor kedua | B (R2), C |
| ether4 | Backbone A | D |
| ether5 | Backbone B | D |
 
* Login pertama CHR: user `admin`, password kosong (Enter). Kalau diminta ganti password atau lisensi, ikuti saja/skip.
* Konfigurasi host (`ip addr add ...`) **hilang saat node di-restart**. Kalau node dimatikan, ketik ulang.
* Kalau `ip addr add` pada host tidak aktif, tambahkan `ip link set eth0 up`.
Pemetaan node:
 
| Node | Appliance | Ruang |
|---|---|---|
| R1 | MikroTik CHR | Ruang Dosen |
| R2 | MikroTik CHR | IF-105 |
| R3 | MikroTik CHR | IF-106 |
| R4 | MikroTik CHR | Ruang Baca (mulai Bagian D) |
| Lungmen | netics-pc | Ruang Dosen |
| Laterano | netics-pc-desktop | Ruang Dosen (host uji utama, ping/mtr) |
| Yan | netics-pc | IF-105 |
| Kazdel | netics-pc | IF-106 |
| Higashi | netics-pc | Ruang Baca (Bagian D) |
 
---
 
## 2. BAGIAN A - Perencanaan di Kertas
 
**Tujuan:** menghasilkan Tabel Alokasi Acuan yang dipakai di seluruh bagian lain. Kalau Bagian A salah, B-C-D ikut salah.
 
### Problem A.1 - Perhitungan VLSM
 
**Apa yang dicari:** prefix, network address, dan broadcast address untuk 9 segmen dari blok `[PFX].20.0/24`.
 
**Cara menghitung:**
 
1. Urutkan dari kebutuhan host terbesar.
2. Cari prefix terbesar yang cukup: host valid = `2^(32-prefix) - 2`.
   * 50 host → `/26` (62) | 40 host → `/26` (62) | 35 host → `/26` (62)
   * 10 host → `/28` (14) | 6 router → `/29` (6) | 2 host → `/30` (2)
3. Alokasikan berurutan tanpa tumpang tindih. Network address harus **kelipatan ukuran blok** (blok /26 = kelipatan 64, /28 = 16, /29 = 8, /30 = 4).
4. Ruang Baca (/26) harus mulai di kelipatan 64. Setelah link terakhir berakhir di `.171`, kelipatan 64 berikutnya yang kosong adalah `.192`. Sisa `.172 - .191` tidak terpakai (cadangan).
**Jawaban (harus cocok dengan Tabel Alokasi Acuan):**
 
| No | Segmen | Kebutuhan | Prefix | Network Address | Broadcast | Host valid |
|---|---|---|---|---|---|---|
| 1 | IF-106 | ±50 | /26 | `[PFX].20.0` | `[PFX].20.63` | .1 - .62 |
| 2 | IF-105 | ±40 | /26 | `[PFX].20.64` | `[PFX].20.127` | .65 - .126 |
| 3 | Ruang Dosen | ±10 | /28 | `[PFX].20.128` | `[PFX].20.143` | .129 - .142 |
| 4 | Backbone A | ≤6 router | /29 | `[PFX].20.144` | `[PFX].20.151` | .145 - .150 |
| 5 | Backbone B | ≤6 router | /29 | `[PFX].20.152` | `[PFX].20.159` | .153 - .158 |
| 6 | Link R1-R2 | 2 | /30 | `[PFX].20.160` | `[PFX].20.163` | .161 - .162 |
| 7 | Link R2-R3 | 2 | /30 | `[PFX].20.164` | `[PFX].20.167` | .165 - .166 |
| 8 | Link R3-R1 | 2 | /30 | `[PFX].20.168` | `[PFX].20.171` | .169 - .170 |
| 9 | Ruang Baca (cadangan) | ±35 | /26 | `[PFX].20.192` | `[PFX].20.255` | .193 - .254 |
 
> Catatan: kalau Ruang Baca ikut diurutkan murni berdasarkan ukuran, ia "seharusnya" berada setelah IF-105. Tetapi soal menyatakan Ruang Baca **cadangan di sisa alokasi**, dan Tabel Alokasi Acuan menetapkan `.192/26`. Ikuti tabel acuan.
 
### Problem A.2 - Rencana Interface dan Routing Table (Chain)
 
**Apa yang dicari:** alamat tiap interface dan route yang harus ada di tiap router (chain R1-R2-R3, link R3-R1 belum dipasang).
 
**Aturan:** ether1 = host pertama blok LAN (jadi default gateway). Di link p2p, host pertama diberikan ke router bernomor lebih kecil.
 
**Tabel Alokasi Interface:**
 
| Router | ether1 (LAN) | ether2 | ether3 |
|---|---|---|---|
| R1 | `[PFX].20.129/28` | `[PFX].20.161/30` | - (belum dipakai) |
| R2 | `[PFX].20.65/26` | `[PFX].20.162/30` | `[PFX].20.165/30` |
| R3 | `[PFX].20.1/26` | `[PFX].20.166/30` | - (belum dipakai) |
 
**Routing Table (rencana):**
 
| Router | Destination | Next Hop | Interface | Metode |
|---|---|---|---|---|
| R1 | Ruang Dosen `.128/28` | - | ether1 | Terhubung Langsung |
| R1 | Link R1-R2 `.160/30` | - | ether2 | Terhubung Langsung |
| R1 | IF-105 `.64/26` | `.162` | ether2 | Static |
| R1 | IF-106 `.0/26` | `.162` | ether2 | Static |
| R2 | IF-105 `.64/26` | - | ether1 | Terhubung Langsung |
| R2 | Link R1-R2 `.160/30` | - | ether2 | Terhubung Langsung |
| R2 | Link R2-R3 `.164/30` | - | ether3 | Terhubung Langsung |
| R2 | Ruang Dosen `.128/28` | `.161` | ether2 | Static |
| R2 | IF-106 `.0/26` | `.166` | ether3 | Static |
| R3 | IF-106 `.0/26` | - | ether1 | Terhubung Langsung |
| R3 | Link R2-R3 `.164/30` | - | ether2 | Terhubung Langsung |
| R3 | Ruang Dosen `.128/28` | `.165` | ether2 | Static |
| R3 | IF-105 `.64/26` | `.165` | ether2 | Static |
 
(Semua next hop ditulis `[PFX].20.xxx`. Route ke subnet link yang tidak terhubung langsung sengaja tidak dipasang karena tidak dibutuhkan untuk komunikasi host-ke-host.)
 
**Peluang summarization:**
 
* **R1:** `.64/26` + `.0/26` keduanya lewat `.162` → bisa digabung menjadi **`[PFX].20.0/25`** (mencakup .0-.127 = tepat IF-106 + IF-105).
* **R2:** tidak bisa (next hop berbeda).
* **R3:** `.128/28` dan `.64/26` tidak membentuk satu blok yang rapi (blok terkecil yang mencakup keduanya akan ikut mencakup IF-106 dan lebih) → **tidak bisa**.
### Problem A.3 - Beban Static Routing
 
**Asumsi:** tiap router punya 1 LAN, semua saling terjangkau, hanya menghitung route ke LAN router lain.
 
* Per router: `n - 1` (satu route per LAN lain).
* Total tanpa backup: `n(n-1)`.
* Dengan backup (tiap tujuan +1 route alternatif): per router `2(n-1)`, total `2n(n-1)`.
* Tambah 1 ruang baru: router baru butuh `n` route, dan **tiap dari n router lama** butuh 1 route baru → **n + 1 router diubah** (semua router, n lama + 1 baru).
| n | Static route per router | Total tanpa cadangan | Total dengan cadangan | Router yang diubah saat +1 ruang |
|---|---|---|---|---|
| 3 | 2 | 6 | 12 | 4 (3 lama + 1 baru) |
| 4 | 3 | 12 | 24 | 5 |
| 10 | 9 | 90 | 180 | 11 |
| n | n-1 | n(n-1) | 2n(n-1) | n+1 |
 
**Kesimpulan yang dibuktikan:** pertumbuhannya kuadratik dan setiap penambahan ruang mengubah **semua** router. Inilah alasan dynamic routing diperlukan.
 
---
 
## 3. BAGIAN B - Static Routing pada Chain Topology
 
**Topologi:** `Laterano/Lungmen - SW - R1 -(e2-e2)- R2 -(e3-e2)- R3 - SW - Kazdel`, dan `Yan - SW - R2 (ether1)`.
 
### Problem B.1 - Membangun Chain dan Mengamati Connected Route
 
**Apa yang dicari:** melihat bahwa setelah memberi IP, router hanya punya *connected route*, sehingga antar ruang belum bisa ping.
 
**Langkah:**
 
1. Di GNS3 tambahkan 3 CHR (masing-masing **5 adapter**) bernama R1, R2, R3, switch tiap ruang, dan host sesuai gambar.
2. Kabel: R1 ether2 ↔ R2 ether2, R2 ether3 ↔ R3 ether2. ether1 tiap router ke switch ruangnya.
3. Pada **setiap router** (ganti R1 sesuai router):
```routeros
/ip dhcp-client remove [find]
/system identity set name=R1
```
 
4. Alamat interface:
```routeros
# ===== R1 =====
/ip address
add address=[PFX].20.129/28 interface=ether1 comment="LAN Ruang Dosen"
add address=[PFX].20.161/30 interface=ether2 comment="link ke R2"
 
# ===== R2 =====
/ip address
add address=[PFX].20.65/26  interface=ether1 comment="LAN IF-105"
add address=[PFX].20.162/30 interface=ether2 comment="link ke R1"
add address=[PFX].20.165/30 interface=ether3 comment="link ke R3"
 
# ===== R3 =====
/ip address
add address=[PFX].20.1/26   interface=ether1 comment="LAN IF-106"
add address=[PFX].20.166/30 interface=ether2 comment="link ke R2"
```
 
5. Host:
```bash
# Lungmen
ip addr add [PFX].20.130/28 dev eth0
ip route add default via [PFX].20.129
 
# Laterano
ip addr add [PFX].20.131/28 dev eth0
ip route add default via [PFX].20.129
 
# Yan
ip addr add [PFX].20.70/26 dev eth0
ip route add default via [PFX].20.65
 
# Kazdel
ip addr add [PFX].20.10/26 dev eth0
ip route add default via [PFX].20.1
```
 
6. Cek di tiap router dan bandingkan dengan baris "Terhubung Langsung" pada rencana A.2:
```routeros
/ip address print
/ip route print
```
 
   Yang diharapkan: hanya route berflag `DAC` (Dynamic, Active, Connected). Tidak ada static.
 
7. **PREDIKSI (tulis dulu):** apakah Laterano bisa menjangkau Yan?
```bash
ping -c 3 [PFX].20.162     # R2 ether2
ping -c 3 [PFX].20.70      # Yan
```
 
**Hasil yang diharapkan & penjelasan:**
 
* Ping ke Yan **gagal** (Laterano akan mendapat *Destination Net Unreachable* dari R1 `.129`, karena R1 tidak punya route ke `.64/26`).
* Ping ke `.162` (alamat R2 di link) **juga gagal**. Request sampai ke R2 karena R1 punya connected route ke `.160/30`, **tetapi R2 tidak punya route kembali ke `.128/28`** sehingga reply tidak bisa pulang. Ini bukti awal pentingnya return route.
* (Sebagai pembanding, ping `[PFX].20.161` (R1 ether2) berhasil karena R1 itu sendiri terhubung langsung.)
**SCREENSHOT:** `/ip address print` (R1, R2, R3), `/ip route print` awal, hasil ping ke `.162`.
 
---
 
### Problem B.2 - Static Route Multi-Hop, Return Path, dan Anatomi Datagram
 
**Apa yang dicari:** (1) pembuktian forward vs return route, (2) path lewat `mtr`, (3) perubahan TTL/Checksum tiap hop di Wireshark.
 
**Langkah 1 - Pasang route di R1 dan R2** (komentar jangan diubah, dipakai oleh perintah selanjutnya):
 
```routeros
# ===== R1 =====
/ip route
add dst-address=[PFX].20.64/26 gateway=[PFX].20.162 comment="IF-105 via R2"
add dst-address=[PFX].20.0/26  gateway=[PFX].20.162 comment="IF-106 via R2"
 
# ===== R2 =====
/ip route
add dst-address=[PFX].20.128/28 gateway=[PFX].20.161 comment="Ruang Dosen via R1"
add dst-address=[PFX].20.0/26   gateway=[PFX].20.166 comment="IF-106 via R3"
```
 
**Langkah 2 - PREDIKSI:** R3 belum punya static route. Apakah ping Laterano → Kazdel berhasil? Reply berhenti di router mana?
 
```bash
# Laterano
ping -c 3 [PFX].20.10
```
```routeros
# R3
/tool traceroute [PFX].20.131
```
 
**Hasil yang diharapkan:**
 
* Ping Laterano → Kazdel **gagal (100% loss)**. Request sampai Kazdel (R1→R2→R3 punya route maju), Kazdel membalas ke `.131` lewat gateway R3, tetapi **R3 tidak punya route ke `.128/28`** → reply **berhenti/dibuang di R3**.
* Traceroute dari R3 ke `.131` gagal sejak awal (no route / tidak ada hop).
* (Bonus bukti: Laterano → Yan sebenarnya **sudah berhasil** sekarang, karena R1 dan R2 sudah punya route pergi dan pulang. Hanya IF-106 yang bergantung pada R3.)
**Langkah 3 - Lengkapi R3, ulangi pengujian:**
 
```routeros
# ===== R3 =====
/ip route
add dst-address=[PFX].20.128/28 gateway=[PFX].20.165 comment="Ruang Dosen via R2"
add dst-address=[PFX].20.64/26  gateway=[PFX].20.165 comment="IF-105 via R2"
```
 
**Langkah 4 - Verifikasi path:**
 
```bash
# Laterano
mtr -r -n -c 5 [PFX].20.70
mtr -r -n -c 5 [PFX].20.10
 
# Kazdel
mtr -r -n -c 5 [PFX].20.131
```
 
**Hasil yang diharapkan:**
 
| Arah | Hop (alamat yang tampil di mtr) |
|---|---|
| Laterano → Yan | `.129` (R1) → `.162` (R2) → `.70` |
| Laterano → Kazdel | `.129` (R1) → `.162` (R2) → `.166` (R3) → `.10` |
| Kazdel → Laterano | `.1` (R3) → `.165` (R2) → `.161` (R1) → `.131` |
 
Alamat hop yang tampil adalah interface router yang **menerima** paket (sisi yang menghadap pengirim), makanya berbeda antara arah pergi dan pulang. Semua 0% loss.
 
**Langkah 5 - Capture Wireshark:**
 
1. Klik kanan link **R1-R2** → *Start capture*. Ulangi untuk link **R2-R3** dan link **switch Ruang Dosen ↔ R1**.
2. Pada tiap Wireshark pasang filter: `icmp`
3. Dari Laterano:
```bash
ping -c 3 [PFX].20.10
```
 
4. Cari **Echo Request yang sama** di ketiga capture (cocokkan dengan nomor `Identification`/`seq`). Lihat bagian *Internet Protocol Version 4*.
**Hasil yang diharapkan:**
 
| Capture | TTL (Echo Request) | Protocol | Header Checksum |
|---|---|---|---|
| LAN Ruang Dosen (SW-R1) | **64** (nilai awal Linux) | ICMP (1) | nilai A |
| Link R1-R2 | **63** | ICMP (1) | nilai B (berbeda) |
| Link R2-R3 | **62** | ICMP (1) | nilai C (berbeda) |
 
Filter yang bisa dipakai: `ip.ttl`, `ip.proto`, `ip.checksum`.
 
**Yang dibuktikan:** tiap router mengurangi TTL 1. **Karena TTL berubah, Header Checksum harus dihitung ulang** sehingga nilainya berbeda di setiap hop, sedangkan Protocol, source, dan destination IP tetap sama. (Alamat MAC juga berubah per hop, bisa disebut sebagai bonus.)
 
**SCREENSHOT:** prediksi, ping + traceroute R3, `/ip route print` ketiga router, 3 hasil mtr, 3 capture (TTL dan checksum).
 
---
 
### Problem B.3 - Manipulasi Route
 
#### (a) Route Summarization
 
**Apa yang dicari:** dua route ke R2 digantikan satu route `/25` tetapi konektivitas tetap sama.
 
```routeros
# ===== R1 =====
/ip route remove [find where comment~"via R2"]
/ip route add dst-address=[PFX].20.0/25 gateway=[PFX].20.162 comment="IF-105 dan IF-106 via R2 (summary)"
/ip route print
```
 
```bash
# Laterano
ping -c 3 [PFX].20.70
ping -c 3 [PFX].20.10
```
 
**Yang dibuktikan:** routing table R1 lebih ringkas (1 route, bukan 2) dan Yan/Kazdel tetap terjangkau. **Summary route ini dipertahankan** sampai Bagian C.
 
#### (b) Longest Prefix Match
 
**PREDIKSI:** setelah ada blackhole `/32` ke Kazdel, mana yang tetap terjangkau: Kazdel (.10), R3 ether1 (.1), Yan (.70)?
 
```routeros
# ===== R1 =====
/ip route add dst-address=[PFX].20.10/32 blackhole comment="uji LPM"
```
```bash
# Laterano
ping -c 3 [PFX].20.10
ping -c 3 [PFX].20.1
ping -c 3 [PFX].20.70
```
 
**Hasil yang diharapkan:**
 
| Tujuan | Route yang menang | Hasil |
|---|---|---|
| `.10` (Kazdel) | `/32` blackhole (paling panjang) | **Gagal** (dibuang) |
| `.1` (R3 ether1) | `/25` via R2 (hanya itu yang cocok) | **Berhasil** |
| `.70` (Yan) | `/25` via R2 | **Berhasil** |
 
Bersihkan setelah selesai:
 
```routeros
/ip route remove [find where comment="uji LPM"]
```
 
**Yang dibuktikan:** entri paling spesifik selalu menang, sehingga satu alamat bisa dijadikan pengecualian tanpa mengubah route utama.
 
#### (c) Routing Loop
 
**Apa yang dicari:** paket berputar R1 ↔ R2 sampai TTL habis.
 
Pastikan capture di link R1-R2 masih berjalan. **PREDIKSI** apa yang terjadi pada paket menuju Kazdel.
 
```routeros
# ===== R2 =====
/ip route set [find where comment="IF-106 via R3"] gateway=[PFX].20.161
```
```bash
# Laterano
ping -c 2 [PFX].20.10
```
 
**Hasil yang diharapkan:**
 
* R1 mengirim ke R2 (route `/25`), R2 mengembalikan ke R1 (gateway `.161`), terus berulang.
* Laterano menerima pesan **ICMP "Time to live exceeded"** (bukan reply). Pesan ini dikirim oleh router tempat TTL habis (R1 atau R2).
* Di capture R1-R2, Echo Request yang sama (ID yang sama) lewat **berulang kali dengan TTL yang turun terus** (kira-kira 63, 62, 61, 60, ... sampai 1), lalu muncul ICMP *Time-to-live exceeded*.
**Kembalikan konfigurasi lalu verifikasi:**
 
```routeros
# ===== R2 =====
/ip route set [find where comment="IF-106 via R3"] gateway=[PFX].20.166
```
```bash
ping -c 3 [PFX].20.10     # harus berhasil lagi
```
 
**Yang dibuktikan:** loop terjadi bila dua router saling menunjuk sebagai next hop. TTL adalah pengaman yang mencegah paket berputar selamanya.
 
**SCREENSHOT (B.3):** route R1 setelah summary, ping setelah summary, hasil 3 ping LPM + prediksi, pesan TTL exceeded, capture dengan TTL menurun, ping setelah dikembalikan.
 
---
 
## 4. BAGIAN C - Redundansi dan OSPF pada Topologi Ring
 
**Pastikan:** akhir Bagian B masih utuh (summary route R1 `/25` tetap ada).
 
### Problem C.1 - Redundant Link dengan Static Routing
 
**Apa yang dicari:** (1) path lebih pendek lewat link R1-R3, (2) asymmetric path, (3) bukti **static route tidak mendeteksi link putus**, (4) perbaikan parsial dengan `check-gateway`.
 
**Langkah 1 - Kabel dan alamat:** sambungkan R1 ether3 ↔ R3 ether3.
 
```routeros
# ===== R1 =====
/ip address add address=[PFX].20.169/30 interface=ether3 comment="link ke R3"
 
# ===== R3 =====
/ip address add address=[PFX].20.170/30 interface=ether3 comment="link ke R1"
```
 
**Langkah 2 - Direct route:**
 
```routeros
# ===== R1 =====
/ip route add dst-address=[PFX].20.0/26 gateway=[PFX].20.170 comment="IF-106 langsung via R3"
 
# ===== R3 =====
/ip route set [find where comment="Ruang Dosen via R2"] distance=10
/ip route add dst-address=[PFX].20.128/28 gateway=[PFX].20.169 comment="Ruang Dosen langsung via R1"
```
 
**Penjelasan:**
 
* R1: route `/26` mengalahkan summary `/25` lewat **Longest Prefix Match**.
* R3: dua route ke `.128/28` punya prefix sama, jadi dipakai **distance**. Route lama diberi `distance=10` supaya jadi cadangan, route baru (distance default 1) jadi utama.
**Langkah 3 - Verifikasi hop:**
 
```bash
mtr -r -n -c 5 [PFX].20.10
```
 
Diharapkan: `.129` (R1) → `.170` (R3) → `.10`. **Lebih pendek** dibanding B.2 (sebelumnya 3 router, sekarang 2).
 
**Langkah 4 - Asymmetric path.** **PREDIKSI** path tiap arah, lalu:
 
```routeros
# ===== R3 =====
/ip route disable [find where comment="Ruang Dosen langsung via R1"]
```
```bash
mtr -r -n -c 5 [PFX].20.10     # dari Laterano
mtr -r -n -c 5 [PFX].20.131    # dari Kazdel
```
```routeros
# ===== R3 (kembalikan) =====
/ip route enable [find where comment="Ruang Dosen langsung via R1"]
```
 
**Hasil yang diharapkan:**
 
| Arah | Path |
|---|---|
| Laterano → Kazdel | `.129` (R1) → `.170` (R3) → `.10` (lewat link langsung) |
| Kazdel → Laterano | `.1` (R3) → `.165` (R2) → `.161` (R1) → `.131` (memutar lewat R2, karena direct route R3 dimatikan dan route cadangan distance 10 aktif) |
 
**Yang dibuktikan:** path pergi dan pulang **tidak harus sama** (asymmetric). Jaringan tetap bekerja selama kedua arah punya route.
 
**Langkah 5 - Uji link failure (static).** **PREDIKSI:** apakah traffic pindah ke R2?
 
```bash
# Laterano
ping -O [PFX].20.10
```
 
Lalu di GNS3: **klik kanan link R1-R3 → Suspend**. Pantau ping (baris `no answer yet for icmp_seq=...`). Di R1:
 
```routeros
/interface print where name=ether3
/ip route print where dst-address=[PFX].20.0/26
```
 
**Hasil yang diharapkan:**
 
* Ping **terus gagal**, traffic **tidak otomatis pindah** ke R2, walau jalur via R2 sebenarnya ada.
* Pada R1, route `/26` via `.170` masih **aktif** dan interface ether3 biasanya masih berstatus running (Suspend di GNS3 membuang paket tetapi tidak mematikan interface). R1 masih "percaya" jalur itu hidup dan terus mengirim ke jalur mati.
**Yang dibuktikan:** **static route tidak punya mekanisme mendeteksi kegagalan**, sehingga redundant link tidak berguna tanpa bantuan lain.
 
**Langkah 6 - check-gateway:** **Resume** link R1-R3, lalu:
 
```routeros
# ===== R1 =====
/ip route set [find where comment="IF-106 langsung via R3"] check-gateway=ping
 
# ===== R3 =====
/ip route set [find where comment="Ruang Dosen langsung via R1"] check-gateway=ping
```
 
Ulangi **Suspend** dengan `ping -O` berjalan. Catat berapa detik ping terputus (hitung jumlah `no answer yet`, tiap ping ≈ 1 detik).
 
**Hasil yang diharapkan:** setelah router mendeteksi gateway tidak menjawab ping (kira-kira belasan detik, catat hasil aktual kamu), route `/26` dinonaktifkan dan traffic jatuh ke `/25` via R2 (R3 jatuh ke route distance 10 via R2). Ping pulih, tetapi **ada downtime** dan konfigurasi tambahan harus dibuat manual di tiap route.
 
**SCREENSHOT:** mtr sebelum/sesudah, mtr asymmetric dua arah, ping saat link suspend, status interface/route R1, downtime dengan check-gateway.
 
---
 
### Problem C.2 - OSPF pada Topologi Ring
 
**Apa yang dicari:** semua static route diganti OSPF, lalu lihat neighbor, route OSPF, dan failover otomatis.
 
**Langkah 1 - Bersihkan static route** (di R1, R2, R3). Pastikan **Resume** link R1-R3 dulu.
 
```routeros
/ip route remove [find where static]
/ip route print
```
 
Diharapkan: hanya connected route (`DAC`).
 
**Langkah 2 - Konfigurasi OSPF:**
 
```routeros
# ===== R1 =====  (R2: router-id=2.2.2.2 ; R3: router-id=3.3.3.3)
/routing ospf instance add name=ospf1 version=2 router-id=1.1.1.1
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template
add area=backbone interfaces=ether2,ether3 type=ptp comment="link inter-router"
add area=backbone interfaces=ether1 passive comment="LAN"
```
 
Catatan per router:
 
* R2 memakai interface yang sama (ether2, ether3 sebagai link, ether1 passive).
* **R3** juga ether2, ether3 (ether3 ke R1, ether2 ke R2).
* `passive` pada ether1: subnet LAN tetap di-advertise tetapi tidak mengirim pesan OSPF ke arah host.
**Langkah 3 - Verifikasi:**
 
```routeros
/routing ospf neighbor print
/routing ospf interface print
/ip route print where ospf
```
 
**Hasil yang diharapkan:**
 
* Neighbor: tiap router punya **2 neighbor** dengan state **Full**.
* Route OSPF (flag `DAo`, distance 110) muncul otomatis untuk subnet LAN dan link yang tidak terhubung langsung.
* R1 menuju IF-106 (`.0/26`) memilih **via R3 langsung** (`.170`) karena cost lebih kecil (kira-kira 20 vs 30 lewat R2, dengan cost default 10 per interface).
Cek: `ping -c 3 [PFX].20.10` dan `ping -c 3 [PFX].20.70` dari Laterano, serta `mtr` untuk melihat path.
 
**Langkah 4 - Link failure dengan OSPF.** **PREDIKSI:** apakah traffic pulih dan berapa lama?
 
```bash
# Laterano
ping -O [PFX].20.10
```
 
GNS3: **Suspend** link R1-R3. Saat suspend, amati di R1:
 
```routeros
/routing ospf neighbor print
/ip route print where dst-address=[PFX].20.0/26
```
 
**Hasil yang diharapkan:**
 
* Traffic **pulih otomatis** lewat R2 **tanpa konfigurasi tambahan**.
* Waktu pemulihan bergantung pada **dead-interval** OSPF (default RouterOS 40 detik, hello 10 detik), karena Suspend GNS3 tidak mematikan interface secara fisik. Catat angka aktual kamu. Kalau interface benar-benar down, pemulihan jauh lebih cepat.
* Neighbor R3 di R1 **hilang** dari daftar (tinggal R2), route `/26` berpindah ke gateway R2 (`.162`) dengan cost lebih besar.
 
**Langkah 5 - Resume** link R1-R3. Setelah neighbor Full kembali, route `/26` kembali ke path langsung (`.170`). Cek dengan `mtr`.
 
**Yang dibuktikan:** OSPF **mendeteksi, menghitung ulang, dan pindah jalur otomatis**, serta kembali ke jalur terbaik saat link pulih. Inilah keunggulan dibanding static routing di C.1.
 
**SCREENSHOT:** neighbor, interface, route OSPF; prediksi + durasi pemulihan; routing table R1 dan neighbor saat suspend; path setelah resume.
 
---
 
## 5. BAGIAN D - Backbone Ganda dan Skalabilitas
 
### Problem D.1 - Migrasi ke Backbone Ganda
 
**Apa yang dicari:** peran DR/BDR di segmen broadcast, ECMP (2 gateway sama cost), dan failover antar backbone.
 
**Langkah 1 - Tambah switch Backbone A dan B.** Sambungkan **ether4 R1/R2/R3 → Backbone A** dan **ether5 R1/R2/R3 → Backbone B**.
 
**Langkah 2 - Konfigurasi di R1, R2, R3:**
 
```routeros
# ===== R1 =====  (R2: .146 dan .154 ; R3: .147 dan .155)
/interface disable ether2,ether3
/ip address
add address=[PFX].20.145/29 interface=ether4 comment="Backbone A"
add address=[PFX].20.153/29 interface=ether5 comment="Backbone B"
/routing ospf interface-template
add area=backbone interfaces=ether4,ether5 comment="backbone ganda"
```
 
Alamat tiap router:
 
| Router | ether4 (Backbone A) | ether5 (Backbone B) |
|---|---|---|
| R1 | `[PFX].20.145/29` | `[PFX].20.153/29` |
| R2 | `[PFX].20.146/29` | `[PFX].20.154/29` |
| R3 | `[PFX].20.147/29` | `[PFX].20.155/29` |
 
**Langkah 3 - DR/BDR:**
 
```routeros
/routing ospf neighbor print
/routing ospf interface print
```
 
**Penjelasan:** DR dipilih berdasarkan **priority tertinggi** (default sama = 128), lalu **router-id tertinggi**. Dengan router-id 1.1.1.1 / 2.2.2.2 / 3.3.3.3, secara teori **R3 = DR, R2 = BDR, R1 = DROther**. Namun DR **non-preemptive**: kalau router yang naik duluan bukan R3, DR bisa berbeda. **Catat apa yang benar-benar terlihat** di layarmu untuk Backbone A dan B.
 
**Langkah 4 - PREDIKSI jumlah gateway R1 ke IF-106, lalu verifikasi:**
 
```routeros
/ip route print detail where dst-address=[PFX].20.0/26
```
 
**Hasil yang diharapkan:** **2 gateway**: `[PFX].20.147` (ether4, Backbone A) dan `[PFX].20.155` (ether5, Backbone B). Cost sama → **ECMP** (load balancing).
 
**Langkah 5 - Failover backbone:**
 
```bash
# Laterano
ping -O [PFX].20.10
```
 
Suspend link **R3 ↔ Backbone A**. Lihat route R1 lagi (`/ip route print detail where dst-address=[PFX].20.0/26`).
 
**Hasil yang diharapkan:** gateway via Backbone A (`.147`) hilang, tersisa via Backbone B (`.155`). Ping bisa terganggu sementara (tergantung flow yang kebetulan melewati jalur yang mati, hingga neighbor dead-interval habis), lalu pulih. Catat durasi downtime. **Resume** link setelah selesai dan pastikan 2 gateway kembali.
 
**Langkah 6 - DR priority:**
 
```routeros
# ===== R1 =====
/routing ospf interface-template set [find where comment="backbone ganda"] priority=0
/interface disable ether4,ether5
/interface enable ether4,ether5
/routing ospf interface print
```
 
**Hasil yang diharapkan:** dengan `priority=0` R1 **tidak boleh menjadi DR maupun BDR**; setelah interface di-bounce, peran R1 menjadi **DROther**. (Kalau sebelum diubah R1 sudah DROther, perubahan tidak terlihat. Itu wajar, jelaskan di laporan. Kalau R1 sebelumnya DR/BDR, peran akan turun dan router lain mengambil alih.)
 
**Yang dibuktikan:** DR/BDR dipilih di segmen broadcast untuk mengurangi jumlah adjacency, bisa diatur lewat priority, dan ECMP memberi redundansi tanpa konfigurasi route manual.
 
**SCREENSHOT:** DR/BDR kedua backbone, route detail R1 (2 gateway), perubahan gateway + downtime saat suspend, interface print sebelum/sesudah priority=0.
 
---
 
### Problem D.2 - Penambahan Ruang Baca lewat R4
 
**Apa yang dicari:** membuktikan **R1, R2, R3 tidak perlu disentuh** untuk menambah ruang baru.
 
**Langkah 1 - Hitung di kertas (static routing + backup lewat backbone kedua):**
 
* R1, R2, R3 masing-masing butuh route ke `.192/26` via R4: primary lewat Backbone A (`.148`) + backup lewat Backbone B (`.156`, distance lebih besar) = 2 route × 3 router = **6 route**.
* R4 butuh route balik ke 3 LAN (`.0/26`, `.64/26`, `.128/28`) × 2 (primary + backup) = **6 route**.
* Total **12 static route di 4 router** (semua router diubah). Dengan OSPF: **0 perubahan pada R1-R3**, hanya R4 dikonfigurasi.
**Langkah 2 - Bangun R4:** tambah CHR (5 adapter), switch Ruang Baca, dan Higashi. R4 ether1 → switch Ruang Baca, ether4 → Backbone A, ether5 → Backbone B.
 
```routeros
# ===== R4 =====
/ip dhcp-client remove [find]
/system identity set name=R4
/ip address
add address=[PFX].20.193/26 interface=ether1 comment="LAN Ruang Baca"
add address=[PFX].20.148/29 interface=ether4 comment="Backbone A"
add address=[PFX].20.156/29 interface=ether5 comment="Backbone B"
/routing ospf instance add name=ospf1 version=2 router-id=4.4.4.4
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template
add area=backbone interfaces=ether4,ether5 comment="backbone ganda"
add area=backbone interfaces=ether1 passive comment="LAN"
```
```bash
# Higashi
ip addr add [PFX].20.200/26 dev eth0
ip route add default via [PFX].20.193
```
 
**Langkah 3 - Verifikasi tanpa menyentuh R1:**
 
```routeros
# ===== R1 =====
/ip route print where dst-address=[PFX].20.192/26
```
```bash
# Laterano
mtr -r -n -c 5 [PFX].20.200
```
 
**Hasil yang diharapkan:** R1 sudah memiliki route `.192/26` hasil OSPF dengan **2 gateway** (`.148` via Backbone A dan `.156` via Backbone B). mtr berhasil menjangkau Higashi: `.129` (R1) → `.148` atau `.156` (R4) → `.200`. R4 tidak mengambil alih DR (non-preemptive), walau router-id-nya paling besar.
 
**Langkah 4 - Simpan konfigurasi sebagai bukti:**
 
```routeros
/export file=case4-R1     # lakukan juga untuk R2, R3, R4 (case4-R2, dst)
/file print
```
 
(Kalau file susah diunduh dari GNS3, screenshot hasil `/export` di terminal dan `/file print`.)
 
**Yang dibuktikan:** pada dynamic routing, **menambah ruang hanya butuh konfigurasi di router baru**. Bandingkan dengan static: 12 route di 4 router.
 
**SCREENSHOT:** perhitungan prediksi, route `.192/26` di R1, mtr Laterano → Higashi, file export 4 router.
 
---
 
## 6. TUGAS MANDIRI (Jawaban Contoh, Kertas)
 
Alokasi `[PFX].50.0/24`: Lab Jaringan (±60), Lab Sistem Cerdas (±25), Ruang Server (±6), chain **RA-RB-RC**. Asumsi: RA = Lab Jaringan, RB = Lab Sistem Cerdas, RC = Ruang Server (cocokkan dengan gambar/asisten bila berbeda).
 
### 1. VLSM
 
| No | Parameter | Lab Jaringan (±60) | Lab Sistem Cerdas (±25) | Ruang Server (±6) | Link RA-RB | Link RB-RC |
|---|---|---|---|---|---|---|
| 1 | Prefix | /26 | /27 | /29 | /30 | /30 |
| 2 | Network | `[PFX].50.0` | `[PFX].50.64` | `[PFX].50.96` | `[PFX].50.104` | `[PFX].50.108` |
| 3 | Broadcast | `.63` | `.95` | `.103` | `.107` | `.111` |
| 4 | Host valid | .1 - .62 | .65 - .94 | .97 - .102 | .105 - .106 | .109 - .110 |
 
Alasan prefix: 60 → /26 (62), 25 → /27 (30), 6 → /29 (6 host, pas), link → /30 (2). Urutan alokasi dari terbesar ke terkecil, tiap network address kelipatan ukuran bloknya.
 
### 2. Rencana interface
 
| Router | ether1 (LAN) | ether2 | ether3 |
|---|---|---|---|
| RA | `[PFX].50.1/26` | `[PFX].50.105/30` | - |
| RB | `[PFX].50.65/27` | `[PFX].50.106/30` | `[PFX].50.109/30` |
| RC | `[PFX].50.97/29` | `[PFX].50.110/30` | - |
 
### 3. Routing table
 
| Router | Destination | Next Hop | Interface | Metode |
|---|---|---|---|---|
| RA | Lab Jaringan `.0/26` | - | ether1 | Terhubung |
| RA | Link RA-RB `.104/30` | - | ether2 | Terhubung |
| RA | Lab Sistem Cerdas `.64/27` | `.106` | ether2 | Static |
| RA | Ruang Server `.96/29` | `.106` | ether2 | Static → **summary** `.64/26` via `.106` |
| RB | Lab Sistem Cerdas `.64/27` | - | ether1 | Terhubung |
| RB | Link RA-RB / RB-RC | - | ether2 / ether3 | Terhubung |
| RB | Lab Jaringan `.0/26` | `.105` | ether2 | Static |
| RB | Ruang Server `.96/29` | `.110` | ether3 | Static |
| RC | Ruang Server `.96/29` | - | ether1 | Terhubung |
| RC | Link RB-RC `.108/30` | - | ether2 | Terhubung |
| RC | Lab Jaringan `.0/26` | `.109` | ether2 | Static |
| RC | Lab Sistem Cerdas `.64/27` | `.109` | ether2 | Static → **summary** `.0/25` via `.109` |
 
Yang bisa di-summarize: **RA** (`.64/27` + `.96/29` → `.64/26`) dan **RC** (`.0/26` + `.64/27` → `.0/25`). **RB tidak bisa** karena next hop berbeda. Catatan: summary RC `.0/25` juga mencakup subnet Ruang Server yang terhubung langsung, tetapi aman karena connected route `/29` lebih spesifik (LPM) sehingga yang menang.
 
### 4. Backup RC-RA
 
Link baru `[PFX].50.112/30` (RA ether3 = `.113`, RC ether3 = `.114`). Tiap router butuh route cadangan (distance lebih tinggi) untuk tiap LAN tujuan lewat arah sebaliknya:
 
* RA: 2 backup (ke Cerdas dan Server via RC `.114`)
* RB: 2 backup (ke Jaringan via RC `.110`, ke Server via RA `.105`)
* RC: 2 backup (ke Jaringan dan Cerdas via RA `.113`)
**Total tambahan: 6 static route** (7 bila menghitung route ke subnet link baru). Tanpa summarization route totalnya menjadi 12 (6 primer + 6 backup).
 
**Alasan dynamic routing:** static tidak mendeteksi link putus (terbukti di C.1), backup harus diatur manual per tujuan dengan distance, dan jumlah route tumbuh `2n(n-1)`. Dynamic routing (OSPF) menghitung path dan failover otomatis, dan menambah ruang hanya memerlukan konfigurasi di router baru (terbukti di D.2).
 
---
 
## 7. Troubleshooting Cepat
 
| Gejala | Cek / Penyebab umum |
|---|---|
| Ping ke router lain gagal padahal route pergi ada | **Return route belum ada** di router tujuan (konsep utama modul ini) |
| Ping host ke gateway gagal | IP host/subnet mask salah, `ip link set eth0 up`, kabel ke switch salah |
| `no route to host` di host | Default gateway belum diset (`ip route add default via ...`) |
| Route static tidak aktif / `X` | Gateway tidak terhubung langsung (alamat gateway harus satu subnet dengan interface) |
| Perintah `[find where comment=...]` tidak jalan | Komentar tidak persis sama. Cek `/ip route print` |
| OSPF neighbor tidak muncul | Interface belum masuk template, subnet mask/IP link beda, area beda, interface disabled, link GNS3 suspend |
| Neighbor stuck di `ExStart/Init` | Mismatch MTU/network-type. Samakan `type=ptp` di kedua sisi link p2p |
| Route OSPF tidak muncul untuk LAN | ether1 belum dimasukkan template (passive) |
| Tidak ada DR/BDR | Interface backbone masih tipe ptp. Template backbone jangan diberi `type=ptp` |
| Ping lewat backbone aneh sebentar | Normal selama convergence/dead-interval 40 detik |
| Capture Wireshark kosong | Capture dimulai setelah ping, atau filter `icmp` salah. Mulai capture dulu baru ping |
| Hasil mtr tidak muncul hop | Gunakan `mtr -r -n -c 5 <ip>`. Firewall/ICMP time-exceeded bisa tidak tampil, coba `ping`/traceroute |
 
Perintah diagnosa umum:
 
```routeros
/ip address print
/ip route print
/ip route print detail
/interface print
/routing ospf neighbor print
/routing ospf interface print
/log print
```
 
---
 
## 8. Cheat Sheet Perintah
 
```routeros
# Alamat
/ip address add address=A.B.C.D/NN interface=etherX comment="..."
# Static route
/ip route add dst-address=NET/NN gateway=GW comment="..."
/ip route add dst-address=IP/32 blackhole
/ip route set [find where comment="..."] gateway=GW distance=10 check-gateway=ping
/ip route disable|enable|remove [find where comment="..."]
/ip route remove [find where static]
# Interface
/interface disable etherX,etherY
/interface enable etherX,etherY
# OSPF
/routing ospf instance add name=ospf1 version=2 router-id=X.X.X.X
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template add area=backbone interfaces=etherX type=ptp
/routing ospf interface-template add area=backbone interfaces=etherX passive
/routing ospf neighbor print
/routing ospf interface print
/ip route print where ospf
# Uji
/ping A.B.C.D
/tool traceroute A.B.C.D
/export file=nama
```
 
```bash
# Host
ip addr add A.B.C.D/NN dev eth0
ip route add default via GW
ping -c 3 IP
ping -O IP                  # tampilkan paket yang belum dibalas (ukur downtime)
mtr -r -n -c 5 IP           # laporan path
```
 
---
 
## 9. Checklist Besok
 
- [ ] Bagian A sudah dihitung dan **cocok dengan Tabel Alokasi Acuan** (`.0/26`, `.64/26`, `.128/28`, `.144/29`, `.152/29`, `.160/30`, `.164/30`, `.168/30`, `.192/26`).
- [ ] `[PFX]` sudah diganti.
- [ ] CHR punya 5 adapter, nama node R1/R2/R3/R4 benar.
- [ ] Semua PREDIKSI ditulis **sebelum** menjalankan perintah.
- [ ] Capture Wireshark dimulai **sebelum** ping.
- [ ] Komentar route persis sama dengan di modul.
- [ ] Setelah manipulasi (LPM, loop, disable route), konfigurasi sudah **dikembalikan**.
- [ ] Sebelum C.2: semua static route dihapus.
- [ ] Bagian D: R1-R3 tidak diubah saat menambah R4.
- [ ] `/export` keempat router tersimpan.
**Ringkasan satu baris tiap bagian:**
A = hitung alamat dan rencana. B = static route butuh forward dan return route, TTL turun tiap hop, LPM menang, loop berhenti karena TTL. C = static tidak tahu link putus, OSPF pulih otomatis. D = broadcast backbone punya DR/BDR dan ECMP, menambah ruang cukup konfigurasi router baru.
