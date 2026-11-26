# Panduan Step-by-Step Pra-Praktikum Modul 4 Jarkom

> IP Addressing, VLSM, Static Routing Multi-Hop, Manipulasi Route, OSPF (Ring & Backbone Ganda), dan Tugas Mandiri.
> Dibuat dari **soal pra-praktikum** + **README Modul 4**. Urutan di sini mengikuti urutan soal (A → B → C → D → E).

---

## Daftar Isi

0. [Baca Ini Dulu (konvensi & alur kerja)](#0-baca-ini-dulu)
1. [Persiapan GNS3 & Cara Konfigurasi Router vs Host](#1-persiapan-gns3--cara-konfigurasi-router-vs-host)
2. [Cheat Sheet: Cara Membaca Output](#2-cheat-sheet-cara-membaca-output)
3. [Bagian A: Perencanaan (kertas)](#3-bagian-a--perencanaan-vlsm-interface-routing-table)
4. [Bagian B: Static Routing Chain](#4-bagian-b--static-routing-pada-chain-topology)
5. [Bagian C: Redundansi & OSPF Ring](#5-bagian-c--redundansi-dan-ospf-pada-topologi-ring)
6. [Bagian D: Backbone Ganda & Skalabilitas](#6-bagian-d--backbone-ganda-dan-skalabilitas)
7. [Bagian E: Tugas Mandiri](#7-bagian-e--tugas-mandiri-kertas)
8. [Troubleshooting Sistematis](#8-troubleshooting-sistematis)
9. [Checklist Screenshot & Hari-H](#9-checklist-screenshot--hari-h)

---

## 0. Baca Ini Dulu

### 0.1 Placeholder IP Prefix

Soal memakai `[IP Prefix].20.0/24`. Di panduan ini aku pakai **contoh `10.139`** (asumsi dari nomor di nama file soalmu).
Jadi alokasi induk = `10.139.20.0/24`.

> **Ganti dengan prefix kamu sendiri.** Paling gampang: tulis semua perintah di Notepad, lalu *Find & Replace* `10.139` → prefix kamu.
> Di praktikum, prefix biasanya ditentukan asisten/tiap praktikan beda, jadi **cek dulu**.

### 0.2 Alur kerja tiap problem (ini yang dinilai!)

Soal bilang: *"Setiap langkah yang meminta prediksi wajib dituliskan terlebih dahulu sebelum perintah dijalankan."*

```
PREDIKSI (tulis + alasan)  →  EKSEKUSI (perintah)  →  OBSERVASI (screenshot)  →  KESIMPULAN (cocok/tidak? kenapa?)
```

Jadi siapkan satu catatan. Tiap ketemu kata **"prediksi"**, tulis dulu jawabanmu + alasannya *sebelum* ngetik perintah. Di panduan ini aku kasih **prediksi + alasan**, tapi pas praktikum tetap **tulis dengan kalimatmu sendiri** dan bandingkan dengan hasil asli.

### 0.3 Peta besar: apa yang dibangun & kenapa

| Bagian | Topologi | Routing | Yang mau dibuktikan |
|---|---|---|---|
| A | Kertas | - | VLSM, rencana alamat, rencana route, beban static routing |
| B | Chain R1–R2–R3 | Static | Forward + return route, TTL/Checksum, summarization, LPM, routing loop |
| C | Ring R1–R2–R3 (+link R3–R1) | Static → OSPF | Static tidak bisa failover otomatis, OSPF bisa |
| D | Backbone A & B (+R4) | OSPF | DR/BDR, ECMP, tambah ruang cuma konfigurasi router baru |
| E | Kertas | - | Latihan A + B versi kecil |

Bagian B → C → D **dibangun di atas topologi yang sama** (jangan hapus project!). Summary route R1 dari B.3(a) **dipertahankan** sampai C.

---

## 1. Persiapan GNS3 & Cara Konfigurasi Router vs Host

### 1.1 Node yang dibutuhkan

| Node | Appliance | Dipakai di |
|---|---|---|
| R1, R2, R3 | MikroTik CHR (**5 network adapter**) | B, C, D |
| R4 | MikroTik CHR (**5 adapter**) | D |
| Lungmen, Yan, Kazdel, Higashi | `netics-pc` | sesuai soal |
| Laterano | `netics-pc-desktop` | B, C, D (pengamat: ping & mtr) |
| Switch1 (Dosen), Switch2 (IF-105), Switch3 (IF-106) | Ethernet switch | B, C, D |
| Switch4 (Ruang Baca), SW-BackboneA, SW-BackboneB | Ethernet switch | D |

> Soal: *"Tambahkan tiga node MikroTik CHR, masing-masing dengan 5 network adapter"*. Default CHR hanya punya sebagian adapter, jadi **wajib diubah**.

### 1.2 Menambah adapter jadi 5 (WAJIB sebelum node dinyalakan)

1. Drag **MikroTik CHR** ke canvas.
2. Pastikan node **dalam keadaan mati** (kalau sudah nyala: `/system shutdown` di console → `y`, atau Stop dari GNS3).
3. Klik kanan node → **Configure** → tab **Network** → ubah **Adapters = 5** → **OK**.
4. Ulangi untuk R1, R2, R3 (dan R4 di Bagian D).

### 1.3 Ganti nama node

Klik kanan node → **Change hostname** (atau Configure → General → Name). Namai persis: `R1, R2, R3, Lungmen, Laterano, Yan, Kazdel`.

> Ini hanya mengganti **label di GNS3**. Hostname di dalam MikroTik tetap `MikroTik` sampai kamu jalankan `/system identity set name=R1` (ada di langkah B.1).

### 1.4 Membuat link (kabel) di GNS3

Ini yang kamu maksud soal *"link to / link via"* pada router: **router tidak bisa langsung "diedit config"** seperti host. Sambungan fisik dibuat dulu lewat tool link, baru IP diatur lewat console.

1. Klik ikon **Add a link** (ikon kabel) di toolbar kiri.
2. Klik node pertama → muncul daftar interface → pilih interface-nya.
3. Klik node kedua → pilih interface-nya.
4. Selesai → klik ikon kabel lagi (atau tombol `Esc`) untuk keluar mode link.

> Kalau nama interface di daftar muncul sebagai `Ethernet0 … Ethernet4`, berarti **Ethernet0 = ether1, Ethernet1 = ether2, … Ethernet4 = ether5**. Cara cek kabelnya benar: setelah node nyala, jalankan `/interface print`. Interface yang kabelnya tersambung akan berflag **R** (running).

**Daftar link Bagian B (chain):**

| Dari | Ke |
|---|---|
| Lungmen `eth0` | Switch1 `e0` |
| Laterano `eth0` | Switch1 `e1` |
| Switch1 `e2` | R1 `ether1` |
| Yan `eth0` | Switch2 `e0` |
| Switch2 `e1` | R2 `ether1` |
| Kazdel `eth0` | Switch3 `e0` |
| Switch3 `e1` | R3 `ether1` |
| R1 `ether2` | R2 `ether2` |
| R2 `ether3` | R3 `ether2` |

(Link tambahan Bagian C dan D ada di bagian masing-masing.)

### 1.5 Router vs Host: cara konfigurasinya BEDA

| | **Router (MikroTik CHR)** | **Host (netics-pc)** |
|---|---|---|
| Klik kanan → *Edit config* | **Tidak ada / tidak dipakai** | **Ada**, mengubah `/etc/network/interfaces` |
| Cara konfigurasi | **Console RouterOS** (CLI) setelah login | Console Linux (`ip addr add`) **atau** Edit config |
| Login | user `admin`, password **kosong** (Enter). Kalau minta ganti password: **Enter / Ctrl+C** | langsung masuk shell |
| Persisten? | Ya, otomatis tersimpan saat restart | `ip addr add`: **hilang saat restart**. *Edit config*: **tetap** |
| IP forwarding | Sudah aktif default (tidak perlu `sysctl`) | Host tidak perlu forwarding |
| Autocomplete | Tekan `TAB` | - |

**Urutan kerja yang benar untuk router:**
1. Buat semua link dulu → 2. Start node → 3. Buka **console** (double-click node) → 4. Login `admin` / Enter → 5. Konfigurasi lewat CLI.

**Host, opsi A (sesuai soal): lewat console**

```bash
ip addr flush dev eth0            # bersihkan IP bawaan (kalau ada DHCP default)
ip addr add 10.139.20.130/28 dev eth0
ip link set eth0 up
ip route add default via 10.139.20.129
```

**Host, opsi B (persisten): Edit config** (klik kanan node → Edit config, lalu **restart node**). Contoh Lungmen:

```
auto eth0
iface eth0 inet static
    address 10.139.20.130
    netmask 255.255.255.240
    gateway 10.139.20.129
```

> Netmask: **/28 = 255.255.255.240**, **/26 = 255.255.255.192**, **/30 = 255.255.255.252**, **/29 = 255.255.255.248**.
> Kalau pakai opsi A, jangan restart host di tengah jalan atau IP-nya hilang. Opsi B lebih aman kalau mau restart.

### 1.6 Konfigurasi awal MikroTik (tiap router, sekali di awal)

```
/ip dhcp-client remove [find]
/system identity set name=R1        # R2, R3, R4 menyesuaikan
```

Kenapa DHCP client dihapus? CHR bawaannya menjalankan DHCP client di `ether1`. Di topologi ini `ether1` adalah **LAN**, bukan ke NAT. Kalau tidak dihapus, router akan "mencari" DHCP server dan bisa muncul route/IP aneh yang bikin routing table tidak sama dengan rencana.

Setelah ini prompt berubah: `[admin@MikroTik] >` → `[admin@R1] >`.

---

## 2. Cheat Sheet: Cara Membaca Output

### 2.1 Flag di `/ip route print`

| Flag | Arti |
|---|---|
| `D` | Dynamic (dibuat otomatis) |
| `A` | Active (route yang dipakai) |
| `c` | Connected (subnet yang terhubung langsung) |
| `s` | Static (dibuat manual) |
| `o` | OSPF |
| `r` | RIP |
| `B` | Blackhole |
| `X` | Disabled |
| `I` | Inactive (misal gateway tidak terjangkau / check-gateway gagal) |

Contoh bentuk output (kurang lebih):

```
[admin@R1] > /ip route print
Flags: D - DYNAMIC; A - ACTIVE; c - CONNECT, s - STATIC, o - OSPF
Columns: DST-ADDRESS, GATEWAY, DISTANCE
#     DST-ADDRESS          GATEWAY       DISTANCE
0 DAc 10.139.20.128/28     ether1               0
1 DAc 10.139.20.160/30     ether2               0
2  As 10.139.20.64/26      10.139.20.162        1
```

Cara baca: **DST-ADDRESS** = subnet tujuan, **GATEWAY** = next hop (atau interface jika connected), **DISTANCE** = prioritas sumber route (connected 0, static 1, OSPF 110; **makin kecil makin dipercaya**).

> Kalau kolom flag-nya `DAc` berarti *connected & active*. Kalau cuma `As` berarti *static & active*. Ini yang kamu bandingkan dengan kolom **Metode** di tabel rencana A.2 ("Terhubung Langsung" vs "Static").

### 2.2 Cara membaca `ping` Linux

```
64 bytes from 10.139.20.10: icmp_seq=1 ttl=61 time=2.1 ms
```

| Bagian | Arti |
|---|---|
| `ttl=61` | Sisa TTL **pada balasan** saat sampai ke kamu |
| `Destination Net Unreachable` (dari IP router) | Router tidak punya route ke tujuan, dan **memberi tahu** kamu |
| Hanya timeout, 100% packet loss, tanpa pesan | Paket hilang diam-diam (tidak ada route balik / blackhole) |
| `Time to live exceeded` | TTL habis → **routing loop** |

**Hitung TTL**: host Linux dan MikroTik mulai dari TTL **64**. Tiap router yang meneruskan paket mengurangi 1.
`ttl_balasan = 64 − jumlah_router_yang_dilewati_balasan`.

### 2.3 Cara membaca `mtr -r -c 5`

`-r` = report (tampil sekali, bukan live), `-c 5` = 5 siklus.

```
HOST: Laterano                Loss%   Snt   Last   Avg  Best  Wrst StDev
  1.|-- 10.139.20.129          0.0%     5    0.5   0.6   0.4   0.9   0.2
  2.|-- 10.139.20.162          0.0%     5    1.0   1.1   0.9   1.5   0.2
  3.|-- 10.139.20.166          0.0%     5    1.4   1.5   1.3   1.9   0.2
  4.|-- 10.139.20.10           0.0%     5    1.9   2.0   1.8   2.3   0.2
```

- Tiap baris = **satu hop** (router, lalu terakhir tujuan).
- **Yang dicari**: jumlah hop, IP tiap hop (urutannya = path), `Loss%`.
- `???` = hop tidak membalas. Kalau mulai satu hop ke bawah semuanya `???`, berarti **paket mati di hop sebelumnya**.
- Router membalas dengan IP interface yang menghadap **balik ke pengirim**. Di chain, itu sama dengan interface masuknya.
- Kalau `mtr` tidak ditemukan di sebuah host: cek `which mtr`, atau pakai `traceroute <ip>` sebagai pengganti.

### 2.4 Cara membaca Wireshark (capture link GNS3)

1. Klik kanan **link** di GNS3 → **Start capture** → Wireshark terbuka.
2. Filter: `icmp` (lalu Enter/Apply).
3. Klik satu paket → di panel detail buka **Internet Protocol Version 4**:
   - `Time to Live` → field `ip.ttl`
   - `Protocol` → `ICMP (1)` → field `ip.proto`
   - `Header Checksum` → field `ip.checksum`
4. Biar gampang dibandingkan: klik kanan field → **Apply as Column**.
5. Filter berguna: `icmp.type == 8` (Echo Request), `icmp.type == 0` (Echo Reply), `icmp.type == 11` (Time Exceeded), `ip.ttl == 63`.
6. **Satu paket yang sama** dikenali lewat `icmp.seq` / `ip.id` yang sama di tiap capture.

### 2.5 Perintah penting MikroTik

| Tujuan | Perintah |
|---|---|
| Lihat IP | `/ip address print` |
| Lihat route | `/ip route print` |
| Status interface | `/interface print` |
| Ping dari router | `/ping 10.139.20.10 count=3` |
| Traceroute dari router | `/tool traceroute 10.139.20.131` (berhenti dengan `Q`) |
| Log | `/log print` |
| Export config | `/export file=nama` / `/export` |

---

## 3. Bagian A: Perencanaan (VLSM, Interface, Routing Table)

> **Tidak butuh GNS3.** Ini dikerjakan di kertas dan jadi acuan semua bagian lain. Hasilnya harus cocok dengan Tabel Alokasi Acuan di Bagian B.

### 3.1 Problem A.1: VLSM untuk seluruh segmen

Alokasi: `10.139.20.0/24`. Ada 8 segmen + Ruang Baca (cadangan, dipakai di Bagian D).

**Langkah 1: urutkan dari kebutuhan host terbesar**

| Urut | Segmen | Butuh |
|---|---|---|
| 1 | IF-106 | ±50 |
| 2 | IF-105 | ±40 |
| 3 | Ruang Dosen | ±10 |
| 4 | Backbone A | hingga 6 router |
| 5 | Backbone B | hingga 6 router |
| 6–8 | Link R1–R2, R2–R3, R3–R1 | 2 |
| (cadangan) | Ruang Baca | ±35 |

**Langkah 2: prefix minimum** (rumus host valid = 2^(32−prefix) − 2)

| Segmen | Butuh | Coba | Host valid | Hasil |
|---|---|---|---|---|
| IF-106 | 50 | /27 = 30 ✗, /26 = 62 ✓ | 62 | **/26** |
| IF-105 | 40 | /27 = 30 ✗, /26 = 62 ✓ | 62 | **/26** |
| Ruang Dosen | 10 | /29 = 6 ✗, /28 = 14 ✓ | 14 | **/28** |
| Backbone A / B | 6 | /29 = 6 ✓ | 6 | **/29** |
| Link point-to-point | 2 | /30 = 2 ✓ | 2 | **/30** |
| Ruang Baca | 35 | /27 = 30 ✗, /26 = 62 ✓ | 62 | **/26** |

**Langkah 3: alokasikan berurutan dari `.0`, tidak tumpang tindih.** Network address harus **kelipatan ukuran blok** (/26 → kelipatan 64, /28 → 16, /29 → 8, /30 → 4).

```
.0   ─ .63    IF-106        /26   (64 alamat)
.64  ─ .127   IF-105        /26
.128 ─ .143   Ruang Dosen   /28   (16)
.144 ─ .151   Backbone A    /29   (8)
.152 ─ .159   Backbone B    /29
.160 ─ .163   Link R1-R2    /30   (4)
.164 ─ .167   Link R2-R3    /30
.168 ─ .171   Link R3-R1    /30
.172 ─ .191   (kosong, tidak terpakai)
.192 ─ .255   Ruang Baca    /26
```

**Langkah 4: Ruang Baca.** Setelah `.171`, blok /26 berikutnya yang *kelipatan 64* adalah `.192`. (`.172` bukan kelipatan 64; `.128/26` sudah terpakai sebagian.) Jadi Ruang Baca = **`10.139.20.192/26`**.

**Tabel jawaban A.1:**

| No | Segmen | Kebutuhan | Prefix | Network Address | Broadcast Address |
|---|---|---|---|---|---|
| 1 | IF-106 | ±50 | /26 | 10.139.20.0 | 10.139.20.63 |
| 2 | IF-105 | ±40 | /26 | 10.139.20.64 | 10.139.20.127 |
| 3 | Ruang Dosen | ±10 | /28 | 10.139.20.128 | 10.139.20.143 |
| 4 | Backbone A | 6 router | /29 | 10.139.20.144 | 10.139.20.151 |
| 5 | Backbone B | 6 router | /29 | 10.139.20.152 | 10.139.20.159 |
| 6 | Link R1–R2 | 2 | /30 | 10.139.20.160 | 10.139.20.163 |
| 7 | Link R2–R3 | 2 | /30 | 10.139.20.164 | 10.139.20.167 |
| 8 | Link R3–R1 | 2 | /30 | 10.139.20.168 | 10.139.20.171 |
| 9 | Ruang Baca | ±35 | /26 | 10.139.20.192 | 10.139.20.255 |

> **Rumus broadcast cepat**: broadcast = network + ukuran blok − 1. Contoh `.128/28`: 128 + 16 − 1 = **143**.
> Cek hasilmu dengan subnet calculator (calculator.net atau jodies.de) seperti di modul 1.3.3.

### 3.2 Problem A.2: Rencana interface & routing table (chain)

Aturan soal: `ether1` = host pertama blok LAN (jadi default gateway). `ether2/ether3` = link; **host pertama link diberikan ke router bernomor lebih kecil**. Link R3–R1 **belum dipasang** di Bagian B.

**Tabel Alokasi Interface:**

| Router | ether1 (LAN) | ether2 | ether3 |
|---|---|---|---|
| R1 | 10.139.20.129/28 | 10.139.20.161/30 (ke R2) | - (belum digunakan) |
| R2 | 10.139.20.65/26 | 10.139.20.162/30 (ke R1) | 10.139.20.165/30 (ke R3) |
| R3 | 10.139.20.1/26 | 10.139.20.166/30 (ke R2) | - (belum digunakan) |

**Cara menyusun routing table (3 langkah per router):**
1. Tulis subnet yang **terhubung langsung** (dari IP interface router itu) → metode *Terhubung Langsung*.
2. Subnet sisanya yang ingin dijangkau → butuh **static route**, isi **next hop** = IP router tetangga pada link yang terhubung langsung.
3. Cari dua route dengan next hop **sama** yang bisa digabung (summarize).

**Routing Table (rencana):**

| Router | Destination Network | Next Hop | Outgoing Interface | Metode |
|---|---|---|---|---|
| R1 | Ruang Dosen 10.139.20.128/28 | - | ether1 | Terhubung Langsung |
| R1 | Link R1–R2 10.139.20.160/30 | - | ether2 | Terhubung Langsung |
| R1 | IF-105 10.139.20.64/26 | 10.139.20.162 | ether2 | Static |
| R1 | IF-106 10.139.20.0/26 | 10.139.20.162 | ether2 | Static |
| R2 | IF-105 10.139.20.64/26 | - | ether1 | Terhubung Langsung |
| R2 | Link R1–R2 10.139.20.160/30 | - | ether2 | Terhubung Langsung |
| R2 | Link R2–R3 10.139.20.164/30 | - | ether3 | Terhubung Langsung |
| R2 | Ruang Dosen 10.139.20.128/28 | 10.139.20.161 | ether2 | Static |
| R2 | IF-106 10.139.20.0/26 | 10.139.20.166 | ether3 | Static |
| R3 | IF-106 10.139.20.0/26 | - | ether1 | Terhubung Langsung |
| R3 | Link R2–R3 10.139.20.164/30 | - | ether2 | Terhubung Langsung |
| R3 | Ruang Dosen 10.139.20.128/28 | 10.139.20.165 | ether2 | Static |
| R3 | IF-105 10.139.20.64/26 | 10.139.20.165 | ether2 | Static |

**Langkah 4: ada yang bisa di-summarize?**

- **R1: YA.** `.64/26` dan `.0/26` keduanya via R2 (`.162`). Keduanya berurutan (0–63 dan 64–127) → digabung jadi **`10.139.20.0/25`** (0–127). Ini yang dipakai di B.3(a).
- **R2: tidak.** Dua route-nya beda next hop (`.161` vs `.166`).
- **R3: tidak rapi.** `.128/28` dan `.64/26` via next hop sama, tetapi tidak berurutan; digabung jadi `.0/24` malah mencakup LAN R3 sendiri dan banyak alamat lain → tidak dianjurkan.

> Syarat summarization (dari modul 2.2.1): subnet **berurutan dalam satu blok** dan **next hop sama**.

### 3.3 Problem A.3: Beban konfigurasi static routing

Asumsi soal: tiap router melayani **1 LAN**, semua router saling terjangkau, tanpa summarization.

**Penalaran:**
1. Satu router harus tahu **semua LAN lain** → **n − 1** route.
2. Total seluruh router → **n × (n − 1)**.
3. Dengan backup: tiap tujuan butuh 2 route → per router **2(n − 1)**, total **2n(n − 1)**.
4. Tambah 1 ruang baru: **semua n router lama** harus ditambah route ke LAN baru (router baru juga dikonfigurasi, tapi ia "router baru", bukan router yang diubah).

| Jumlah Router (n) | Static Route per Router | Total Tanpa Cadangan | Total Dengan Cadangan | Router yang Diubah saat Menambah 1 Ruang |
|---|---|---|---|---|
| 3 | 2 | 6 | 12 | 3 |
| 4 | 3 | 12 | 24 | 4 |
| 10 | 9 | 90 | 180 | 10 |
| n | n − 1 | n(n − 1) | 2n(n − 1) | n |

> Pesan intinya: bebannya tumbuh **kuadrat** (n²) dan tiap penambahan menyentuh **semua** router. Ini alasan kita butuh *dynamic routing* (OSPF): router baru cukup dikonfigurasi sendiri.

---

## 4. Bagian B: Static Routing pada Chain Topology

### 4.1 Tabel Alokasi Acuan (isi dari hasil A.1 & A.2)

| Segmen | Blok VLSM | Alamat yang Digunakan |
|---|---|---|
| IF-106 | /26 (10.139.20.0) | R3 (ether1): 10.139.20.1 ; Kazdel (eth0): 10.139.20.10 |
| IF-105 | /26 (10.139.20.64) | R2 (ether1): 10.139.20.65 ; Yan (eth0): 10.139.20.70 |
| Ruang Dosen | /28 (10.139.20.128) | R1 (ether1): .129 ; Lungmen: .130 ; Laterano: .131 |
| Backbone A | /29 (10.139.20.144) | R1 .145, R2 .146, R3 .147, R4 .148 (semua **ether4**) |
| Backbone B | /29 (10.139.20.152) | R1 .153, R2 .154, R3 .155, R4 .156 (semua **ether5**) |
| Link R1–R2 | /30 (10.139.20.160) | R1 (ether2) .161 ; R2 (ether2) .162 |
| Link R2–R3 | /30 (10.139.20.164) | R2 (ether3) .165 ; R3 (ether2) .166 |
| Link R3–R1 | /30 (10.139.20.168) | R1 (ether3) .169 ; R3 (ether3) .170 |
| Ruang Baca | /26 (10.139.20.192) | R4 (ether1) .193 ; Higashi (eth0) .200 |

> Semua `.x` di atas berarti `10.139.20.x`.

### 4.2 Problem B.1: Bangun chain & amati connected route

**Langkah 1–2: bangun topologi.** Tambah 3 CHR (5 adapter, lihat [1.2](#12-menambah-adapter-jadi-5-wajib-sebelum-node-dinyalakan)), 3 switch, 4 host. Buat link sesuai [daftar link 1.4](#14-membuat-link-kabel-di-gns3). Start semua node.

**Langkah 3: login & reset awal tiap router** (R1 contoh; R2, R3 sesuaikan nama):

```
/ip dhcp-client remove [find]
/system identity set name=R1
```

**Langkah 4: IP interface**

```
# ===== R1 =====
/ip address
add address=10.139.20.129/28 interface=ether1 comment="LAN Ruang Dosen"
add address=10.139.20.161/30 interface=ether2 comment="link ke R2"

# ===== R2 =====
/ip address
add address=10.139.20.65/26  interface=ether1 comment="LAN IF-105"
add address=10.139.20.162/30 interface=ether2 comment="link ke R1"
add address=10.139.20.165/30 interface=ether3 comment="link ke R3"

# ===== R3 =====
/ip address
add address=10.139.20.1/26   interface=ether1 comment="LAN IF-106"
add address=10.139.20.166/30 interface=ether2 comment="link ke R2"
```

> Tips: tulis `/ip address` sekali, lalu baris `add ...` di bawahnya berjalan di dalam menu itu.
> Kalau salah IP: `/ip address remove [find address="10.139.20.129/28"]` lalu tambah lagi.
> Cek: `/ip address print` → kolom **NETWORK** harus sama dengan Network Address hasil A.1. Kalau beda, berarti prefix/IP salah ketik.

**Langkah 5: IP host + default gateway**

```bash
# Lungmen
ip addr add 10.139.20.130/28 dev eth0
ip route add default via 10.139.20.129

# Laterano
ip addr add 10.139.20.131/28 dev eth0
ip route add default via 10.139.20.129

# Yan
ip addr add 10.139.20.70/26 dev eth0
ip route add default via 10.139.20.65

# Kazdel
ip addr add 10.139.20.10/26 dev eth0
ip route add default via 10.139.20.1
```

(Kalau host sudah punya IP bawaan: `ip addr flush dev eth0` dulu. Verifikasi: `ip a show eth0` dan `ip route`.)

**Langkah 6: routing table tiap router, bandingkan dengan rencana A.2**

```
/ip route print
```

Hasil yang diharapkan: **hanya route `DAc`** (connected).
- R1: `.128/28` (ether1) dan `.160/30` (ether2)
- R2: `.64/26` (ether1), `.160/30` (ether2), `.164/30` (ether3)
- R3: `.0/26` (ether1), `.164/30` (ether2)

Itu persis baris **"Terhubung Langsung"** di tabel A.2. Tidak ada `0.0.0.0/0` (karena DHCP client sudah dihapus). Kalau masih ada route lain, berarti ada konfigurasi yang tersisa.

**Langkah 7: PREDIKSI sebelum ping**

> **Prediksi (isi sendiri dulu):** apakah Laterano dapat menjangkau Yan?

Jawaban + alasan:
- **Laterano → Yan (`.70`): GAGAL.** R1 hanya tahu subnet connected. R1 **tidak punya route** ke `10.139.20.64/26`, jadi R1 membalas **`Destination Net Unreachable`** ke Laterano.
- **Laterano → `.162` (R2): GAGAL (100% loss, tanpa pesan error).** Ini jebakan yang menarik. Paket **pergi** sampai: Laterano → R1 → R2, karena `.160/30` terhubung langsung di R1. Tapi balasan R2 menuju `.131` (Ruang Dosen) tidak punya jalan: R2 tidak punya route ke `.128/28`. Balasan dibuang diam-diam.

```bash
# dari Laterano
ping -c 3 10.139.20.162     # alamat R2 pada link R1-R2
ping -c 3 10.139.20.70      # Yan
```

> Bukti bahwa paket pergi **sebenarnya sampai**: dari R1 jalankan `/ping 10.139.20.162 count=3`. Ini **berhasil**, karena R2 punya route balik ke `.161` (connected, `.160/30`). Perbedaan "dari Laterano gagal, dari R1 berhasil" membuktikan masalahnya ada di **return route**.

**Tabel observasi B.1** (screenshot):

| No | Parameter | Perintah | Yang discreenshot |
|---|---|---|---|
| 1 | Alamat interface R1, R2, R3 | `/ip address print` | Output 3 router |
| 2 | Routing table awal | `/ip route print` | Hanya `DAc` |
| 3 | Hasil ping ke R2 (.162) | `ping` | 100% loss dari Laterano (+ ping dari R1 berhasil sebagai pembanding) |

### 4.3 Problem B.2: Static route multi-hop, forward/return, anatomi datagram

Prinsip: untuk komunikasi A↔B, **setiap router di jalur pergi harus tahu cara ke B**, dan **setiap router di jalur pulang harus tahu cara ke A**. Satu saja kurang → gagal.

**Langkah 1: static route di R1 dan R2** (R3 *sengaja* belum)

```
# ===== R1 =====
/ip route
add dst-address=10.139.20.64/26 gateway=10.139.20.162 comment="IF-105 via R2"
add dst-address=10.139.20.0/26  gateway=10.139.20.162 comment="IF-106 via R2"

# ===== R2 =====
/ip route
add dst-address=10.139.20.128/28 gateway=10.139.20.161 comment="Ruang Dosen via R1"
add dst-address=10.139.20.0/26   gateway=10.139.20.166 comment="IF-106 via R3"
```

Penjelasan parameter: `dst-address` = subnet tujuan, `gateway` = **next hop** (IP router tetangga pada link yang terhubung langsung). Gateway wajib berada di subnet yang terhubung langsung.

**Langkah 2: PREDIKSI.** R3 belum punya static route. Apakah ping Laterano → Kazdel berhasil? Reply berhenti di router mana?

Jawaban: **GAGAL.** Request sampai: Laterano → R1 → R2 → R3 → Kazdel (R1 punya route `.0/26`, R2 punya route `.0/26`, R3 connected). Kazdel membalas ke `.131`, default gateway-nya R3, tapi **R3 tidak punya route ke `.128/28`**, jadi **reply berhenti di R3**.

```bash
# dari Laterano
ping -c 3 10.139.20.10
```
```
# di R3
/tool traceroute 10.139.20.131
```

Hasil traceroute R3 → Laterano: **tidak ada hop yang muncul** (kosong/timeout), karena R3 sendiri tidak punya route menuju `.131`. Itu konfirmasi: titik putusnya di R3.

**Langkah 3: lengkapi R3**

```
# ===== R3 =====
/ip route
add dst-address=10.139.20.128/28 gateway=10.139.20.165 comment="Ruang Dosen via R2"
add dst-address=10.139.20.64/26  gateway=10.139.20.165 comment="IF-105 via R2"
```

Ulangi `ping -c 3 10.139.20.10` dari Laterano → sekarang **berhasil**.

**Langkah 4: verifikasi path dengan mtr**

```bash
# dari Laterano
mtr -r -c 5 10.139.20.70     # ke Yan
mtr -r -c 5 10.139.20.10     # ke Kazdel

# dari Kazdel
mtr -r -c 5 10.139.20.131    # ke Laterano
```

Hasil yang diharapkan:

| Dari → Ke | Hop yang muncul (urut) | Jumlah hop |
|---|---|---|
| Laterano → Yan | R1 `.129` → R2 `.162` → Yan `.70` | 3 |
| Laterano → Kazdel | R1 `.129` → R2 `.162` → R3 `.166` → Kazdel `.10` | 4 |
| Kazdel → Laterano | R3 `.1` → R2 `.165` → R1 `.161` → Laterano `.131` | 4 |

Jalur pergi dan pulang sama (simetris), hanya urutannya terbalik.

**Langkah 5–6: capture Wireshark & amati field IPv4**

1. Klik kanan link **R1–R2** → *Start capture*. Ulangi untuk link **R2–R3** dan link **Switch1–R1** (switch Ruang Dosen ke R1). Total 3 jendela Wireshark.
2. Di tiap jendela, filter: `icmp`.
3. Dari Laterano: `ping -c 3 10.139.20.10`.
4. Di ketiga capture, ambil **Echo Request yang sama** (seq sama) dan catat 3 field.

Hasil yang diharapkan (Echo Request Laterano → Kazdel):

| Capture | `ip.ttl` Echo Request | `ip.proto` | `ip.checksum` |
|---|---|---|---|
| LAN Ruang Dosen (Switch1–R1) | **64** | 1 (ICMP) | nilai A |
| Link R1–R2 | **63** | 1 | nilai B (≠ A) |
| Link R2–R3 | **62** | 1 | nilai C (≠ B) |

Penjelasan:
- **TTL berkurang 1 tiap router.** Di LAN belum dilewati router mana pun (64), sesudah R1 jadi 63, sesudah R2 jadi 62. Sampai ke Kazdel: 61.
- **Protocol tetap 1 (ICMP)** di semua link, karena yang dibawa tidak berubah.
- **Header Checksum berubah di tiap hop**, karena TTL ada di dalam header; TTL berubah → checksum header **dihitung ulang** router.
- Echo Reply (arah balik) juga bisa dilihat: Kazdel mulai 64; di link R2–R3 = 63, link R1–R2 = 62, LAN Dosen = 61. Ini sama dengan `ttl=61` di output `ping` Laterano.
- Kalau Wireshark menandai checksum "unverified" pada paket yang baru keluar dari host, itu normal (checksum offload di interface virtual). Yang penting nilainya **berbeda** antar link.

**Tabel observasi B.2:**

| No | Parameter | Perintah | Yang discreenshot / ditulis |
|---|---|---|---|
| 1 | Prediksi ping ke Kazdel sebelum R3 dikonfigurasi | - | Tulisan prediksi + alasan |
| 2 | Hasil ping & traceroute dari R3 | `ping`, `/tool traceroute` | Ping gagal + traceroute R3 kosong |
| 3 | Routing table R1, R2, R3 lengkap | `/ip route print` | 3 router, ada `As` |
| 4 | Path Ruang Dosen → IF-105 | `mtr` | 3 hop |
| 5 | Path Ruang Dosen → IF-106 | `mtr` | 4 hop |
| 6 | Path IF-106 → Ruang Dosen | `mtr` | dari Kazdel, 4 hop |
| 7 | TTL di LAN Dosen, link R1–R2, R2–R3 | `ip.ttl` | 64, 63, 62 |
| 8 | Protocol & Header Checksum di ketiga capture | `ip.proto`, `ip.checksum` | 1 dan checksum beda-beda |

### 4.4 Problem B.3: Manipulasi route

Setiap manipulasi: **prediksi → eksekusi → amati → kembalikan konfigurasi**.

#### (a) Route Summarization

```
# ===== R1 =====
/ip route remove [find where comment~"via R2"]
/ip route add dst-address=10.139.20.0/25 gateway=10.139.20.162 comment="IF-105 dan IF-106 via R2 (summary)"
/ip route print
```

Penjelasan: `comment~"via R2"` = hapus semua route yang komentarnya mengandung "via R2" (dua route lama). `10.139.20.0/25` mencakup `.0`–`.127` = IF-106 (`.0/26`) + IF-105 (`.64/26`).

Hasil `/ip route print` R1: route static kini **satu** (`.0/25`), plus 2 connected.

Verifikasi: dari Laterano `ping -c 3 10.139.20.70` (Yan) dan `ping -c 3 10.139.20.10` (Kazdel): **tetap berhasil**.
**Summary route ini dipertahankan sampai Bagian C.**

#### (b) Longest Prefix Match

**Langkah 1: PREDIKSI.** R1 ditambah host route `/32` ke Kazdel bertipe blackhole. Mana yang tetap terjangkau: Kazdel (`.10`), R3 ether1 (`.1`), Yan (`.70`)?

```
# ===== R1 =====
/ip route add dst-address=10.139.20.10/32 blackhole comment="uji LPM"
```

Jawaban dan alasan:

| Tujuan | Route yang cocok di R1 | Dipilih (prefix terpanjang) | Hasil |
|---|---|---|---|
| Kazdel `.10` | `.0/25` dan `.10/32` | **/32 blackhole** | **GAGAL** (dibuang diam-diam) |
| R3 ether1 `.1` | `.0/25` saja | `.0/25` via R2 | **BERHASIL** |
| Yan `.70` | `.0/25` saja | `.0/25` via R2 | **BERHASIL** |

Kenapa `.10` gagal tapi `.1` berhasil padahal sama-sama di dalam `.0/26`? Router memilih route dengan **prefix terpanjang** (paling spesifik). `/32` hanya mencakup 1 alamat; alamat lain tetap ikut route `/25`.

```bash
# dari Laterano
ping -c 3 10.139.20.10
ping -c 3 10.139.20.1
ping -c 3 10.139.20.70
```

> **Catatan:** `blackhole` membuang paket **tanpa memberi tahu** pengirim (tidak ada pesan "unreachable"). Jadi `.10` hanya timeout/100% loss.

**Langkah 3: hapus route uji**

```
/ip route remove [find where comment="uji LPM"]
```

Ulangi ping ke `.10`: kembali berhasil.

#### (c) Routing Loop

**Langkah 1: PREDIKSI.** Pastikan capture di link R1–R2 **masih berjalan** (filter `icmp`). Di R2, arahkan route IF-106 kembali ke R1.

```
# ===== R2 =====
/ip route set [find where comment="IF-106 via R3"] gateway=10.139.20.161
```

Prediksi dan alasan: Laterano → Kazdel. R1 (summary `.0/25`) kirim ke R2. R2 sekarang kirim `.0/26` **balik ke R1** (`.161`). R1 kirim ke R2 lagi. **Paket berputar R1 ↔ R2** sampai TTL habis; pengirim menerima **`Time to live exceeded`**.

```bash
# dari Laterano
ping -c 2 10.139.20.10
```

Output yang diharapkan:
```
From 10.139.20.162 icmp_seq=1 Time to live exceeded
```
Router yang mengirim pesan itu adalah router tempat TTL **habis**. Dengan TTL awal 64 (genap), R1 menerima nilai genap dan R2 menerima nilai ganjil sampai TTL = 1, jadi **R2 (`.162`)** yang biasanya mengirimnya. **Baca IP yang benar-benar muncul di outputmu**, itu yang dicatat.

**Langkah 3: capture link R1–R2.** Filter `icmp`. Kamu akan melihat **Echo Request yang sama (seq sama) lewat berulang kali** dengan TTL terus turun: R1→R2 membawa 63, 61, 59…; R2→R1 membawa 62, 60, 58…; sampai akhirnya muncul **ICMP Type 11 (Time-to-live exceeded)**. Filter `icmp.type == 11` untuk menemukannya.

**Langkah 4: kembalikan**

```
# ===== R2 =====
/ip route set [find where comment="IF-106 via R3"] gateway=10.139.20.166
```

Ulangi `ping -c 3 10.139.20.10`: kembali berhasil.

**Tabel observasi B.3:**

| No | Parameter | Perintah | Yang discreenshot |
|---|---|---|---|
| 1 | Routing table R1 setelah summarization | `/ip route print` | 1 static `.0/25` |
| 2 | Konektivitas ke IF-105 dan IF-106 setelah summarization | `ping` | Yan & Kazdel berhasil |
| 3 | Prediksi dan hasil uji LPM (.10, .1, .70) | `ping` | `.10` gagal; `.1`, `.70` berhasil |
| 4 | Message yang diterima pengirim saat loop | `ping` | `Time to live exceeded` |
| 5 | Nilai TTL Echo Request yang sama pada capture R1–R2 | `ip.ttl` | Deretan TTL menurun |
| 6 | Konektivitas setelah konfigurasi dikembalikan | `ping` | Berhasil |

---

## 5. Bagian C: Redundansi dan OSPF pada Topologi Ring

> Prasyarat: topologi akhir Bagian B, **summary route R1 tetap terpasang**.

### 5.1 Problem C.1: Redundant link dengan static routing

**Langkah 1: tambah link R3–R1.** Di GNS3, link **R1 `ether3` ↔ R3 `ether3`** (boleh saat node menyala). Beri IP:

```
# ===== R1 =====
/ip address add address=10.139.20.169/30 interface=ether3 comment="link ke R3"

# ===== R3 =====
/ip address add address=10.139.20.170/30 interface=ether3 comment="link ke R1"
```

Cek `/ip route print`: muncul connected baru `.168/30`.

**Langkah 2: direct route di kedua sisi**

```
# ===== R1 =====
/ip route add dst-address=10.139.20.0/26 gateway=10.139.20.170 comment="IF-106 langsung via R3"

# ===== R3 =====
/ip route set [find where comment="Ruang Dosen via R2"] distance=10
/ip route add dst-address=10.139.20.128/28 gateway=10.139.20.169 comment="Ruang Dosen langsung via R1"
```

Kenapa beda perlakuan?
- **R1:** route baru `/26` lebih spesifik dari summary `/25` → menang lewat **longest prefix match**. Tidak perlu atur distance.
- **R3:** route lama dan baru **sama-sama `.128/28`** (prefix sama, LPM tidak bisa memilih). Dibedakan lewat **distance**: route lama diberi `distance=10` (kurang dipercaya) → jadi **backup**; route baru distance default `1` → jadi utama.

**Langkah 3: verifikasi path**

```bash
# dari Laterano
mtr -r -c 5 10.139.20.10
```

Hasil: R1 `.129` → R3 `.170` → Kazdel `.10` = **3 hop**. Di B.2 jumlahnya **4 hop** (lewat R2). Link langsung memperpendek jalur.

**Langkah 4: asymmetric path**

**PREDIKSI sebelum menjalankan.** R3 tidak boleh memakai direct route ke Ruang Dosen:

```
# ===== R3 =====
/ip route disable [find where comment="Ruang Dosen langsung via R1"]
```
```bash
mtr -r -c 5 10.139.20.10      # dari Laterano
mtr -r -c 5 10.139.20.131     # dari Kazdel
```
```
# ===== R3 ===== (kembalikan)
/ip route enable [find where comment="Ruang Dosen langsung via R1"]
```

Jawaban dan alasan:

| Arah | Path | Hop |
|---|---|---|
| Laterano → Kazdel | R1 → **R3** (R1 masih punya route `/26` via `.170`) → Kazdel | 3 |
| Kazdel → Laterano | R3 → **R2** → R1 → Laterano (R3 sekarang hanya punya route backup distance 10 via R2) | 4 |

Jalur pergi dan pulang **berbeda** → **asymmetric path**. Berhasil, tapi jalurnya tidak simetris karena routing dikonfigurasi per router per arah.

**Langkah 5: uji link failure (static)**

**PREDIKSI.** Apakah traffic pindah lewat R2 saat link R1–R3 di-suspend?

```bash
# dari Laterano, ping terus-menerus
ping -O 10.139.20.10
```

`-O` membuat ping menampilkan `no answer yet for icmp_seq=N` setiap ada paket yang belum terbalas, jadi putusnya traffic kelihatan jelas. Lalu di GNS3: **klik kanan link R1–R3 → Suspend**.

Jawaban dan alasan: **TIDAK, traffic tidak berpindah.** Static route tidak punya mekanisme mendeteksi kegagalan; ia hanya melihat apakah *interface* hidup. Di GNS3, *Suspend* menahan paket di kabel virtual, tetapi interface router **tetap dianggap running**. Route direct tetap `Active`, router terus mengirim ke link mati. Ping berhenti dan **tidak pulih sendiri** (meski backup route lewat R2 sebenarnya ada).

**Langkah 6: periksa saat link suspend**

```
# ===== R1 =====
/interface print where name=ether3
/ip route print where dst-address=10.139.20.0/26
```

Yang diamati: `ether3` **masih berflag `R` (running)**, dan route `.0/26` via `.170` **masih Active**. Router tidak tahu link mati. Itulah penyebabnya.

**Langkah 7: Resume + check-gateway**

Di GNS3: klik kanan link → **Resume**. Lalu:

```
# ===== R1 =====
/ip route set [find where comment="IF-106 langsung via R3"] check-gateway=ping

# ===== R3 =====
/ip route set [find where comment="Ruang Dosen langsung via R1"] check-gateway=ping
```

`check-gateway=ping` membuat router mem-ping gateway secara berkala; jika tidak terjawab, route ditandai **Inactive** sehingga route berikutnya (backup) dipakai.

Ulangi `ping -O 10.139.20.10` dari Laterano + **Suspend** link R1–R3. Kali ini ping **pulih sendiri setelah jeda** (kira-kira belasan detik; **catat durasimu sendiri**: hitung jumlah `no answer yet` × 1 detik). Saat sudah pindah: R1 route `.0/26` via `.170` jadi **Inactive** dan R1 memakai summary `.0/25` via R2; R3 memakai route backup distance 10 via R2. Path jadi R1 → R2 → R3 → Kazdel.

> Kenapa **kedua sisi** perlu `check-gateway`? Jalur pergi dan jalur pulang masing-masing butuh deteksi. Kalau hanya satu sisi, sisi lain masih mengirim ke link mati.

**Setelah selesai:** Resume link, biarkan pulih.

> Kesimpulan C.1: static routing + check-gateway bisa failover, tapi harus dikonfigurasi **manual per route** dan detiknya lambat. OSPF mengotomatiskannya.

### 5.2 Problem C.2: OSPF pada topologi ring

**Langkah 1: hapus semua static route**

```
# di R1, R2, R3
/ip route remove [find where static]
/ip route print
```

Hasil: hanya route `DAc` (connected). PC antar-ruang kembali tidak saling jangkau, itu normal.

**Langkah 2: konfigurasi OSPF** (3 bagian berurutan: **instance → area → interface-template**)

```
# ===== R1 =====  (R2: router-id=2.2.2.2, R3: router-id=3.3.3.3)
/routing ospf instance add name=ospf1 version=2 router-id=1.1.1.1
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template
add area=backbone interfaces=ether2,ether3 type=ptp comment="link inter-router"
add area=backbone interfaces=ether1 passive comment="LAN"
```

Per router:

| Router | router-id | interface `ptp` | interface `passive` |
|---|---|---|---|
| R1 | 1.1.1.1 | ether2, ether3 | ether1 |
| R2 | 2.2.2.2 | ether2, ether3 | ether1 |
| R3 | 3.3.3.3 | ether2, ether3 | ether1 |

Penjelasan:
- `type=ptp` (point-to-point): link cuma 2 router → **tidak ada pemilihan DR/BDR**, lebih cepat.
- `passive`: subnet LAN **tetap di-advertise** ke router lain, tetapi **tidak mengirim Hello** ke arah host (tidak ada router OSPF di sana).
- Tidak ada `redistribute=connected`: OSPF otomatis mengumumkan subnet semua interface yang ada di template.

**Langkah 3: verifikasi**

```
/routing ospf neighbor print
/routing ospf interface print
/ip route print where ospf
```

Yang dicari:
- **Neighbor**: tiap router punya **2 neighbor** (R1: R2 & R3, dst.) dengan state **`Full`**. Kalau state macet di `Init`/`ExStart`, ada yang salah (lihat [Troubleshooting](#8-troubleshooting-sistematis)).
- **Interface**: ether2/ether3 tipe ptp, ether1 `passive`.
- **Route OSPF**: flag `o`, distance **110**. Contoh di R1: `.0/26` (IF-106) via R3 `.170` (cost total lebih kecil), `.64/26` (IF-105) via R2 `.162`. Route ke `.164/30` kemungkinan punya dua gateway (R2 dan R3, cost sama = *equal cost*).

Cek juga ping antar ruang: semua kembali berhasil **tanpa satu pun static route**.

**Langkah 4: uji failure pada OSPF**

**PREDIKSI.** Traffic pulih? Berapa lama?

```bash
ping -O 10.139.20.10      # dari Laterano
```
Lalu **Suspend** link R1–R3 di GNS3. Amati sambil jalan:

```
# ===== R1 =====
/routing ospf neighbor print
/ip route print where dst-address=10.139.20.0/26
```

Jawaban dan alasan: **Pulih sendiri, tanpa konfigurasi.** Karena interface tetap running (Suspend GNS3), router **tidak langsung tahu** link putus. Ia baru menyimpulkannya saat **Hello tidak terdengar selama dead interval = 40 detik** (hello 10 detik, dead 4×hello). Jadi **downtime kira-kira 40 detik** (catat angka aslimu). Setelah neighbor R3 dianggap mati: neighbor R3 **hilang dari daftar**, LSA baru disebarkan, dan R1 menghitung ulang: route ke `.0/26` berpindah ke gateway R2 (`.162`). Path jadi R1 → R2 → R3 → Kazdel.

**Langkah 5: Resume** link R1–R3. Dalam beberapa detik neighbor R3 muncul lagi `Full`, dan route ke `.0/26` kembali **langsung via R3** (cost lebih kecil). Cek `mtr` lagi: kembali 3 hop.

**Perbandingan (isi sesuai hasil aslimu):**

| | Static (tanpa check-gateway) | Static + check-gateway | OSPF |
|---|---|---|---|
| Pindah ke backup otomatis? | Tidak | Ya (per route) | Ya |
| Waktu pulih | Tidak pernah | belasan detik (catat) | ±40 detik di GNS3 Suspend (catat) |
| Konfigurasi | Manual | Manual | Otomatis |

> Catatan: Di modul, failover OSPF ±1 detik karena interface **di-disable** lewat CLI (router langsung tahu). Di soal ini yang dipakai **Suspend**, sehingga yang bekerja adalah deteksi lewat dead interval. Jangan bandingkan dua angka itu langsung.

---

## 6. Bagian D: Backbone Ganda dan Skalabilitas

> Prasyarat: Bagian C selesai, OSPF berfungsi.

### 6.1 Problem D.1: Migrasi ke backbone ganda

**Langkah 1: tambah switch.** Tambah **SW-BackboneA** dan **SW-BackboneB**. Sambungkan:

| Dari | Ke |
|---|---|
| R1 `ether4` | SW-BackboneA `e0` |
| R2 `ether4` | SW-BackboneA `e1` |
| R3 `ether4` | SW-BackboneA `e2` |
| R1 `ether5` | SW-BackboneB `e0` |
| R2 `ether5` | SW-BackboneB `e1` |
| R3 `ether5` | SW-BackboneB `e2` |

(Port `e3` kedua switch disiapkan untuk R4 di D.2.)

**Langkah 2: matikan link ptp, beri IP backbone, masukkan ke OSPF**

```
# ===== R1 =====
/interface disable ether2,ether3
/ip address
add address=10.139.20.145/29 interface=ether4 comment="Backbone A"
add address=10.139.20.153/29 interface=ether5 comment="Backbone B"
/routing ospf interface-template
add area=backbone interfaces=ether4,ether5 comment="backbone ganda"

# ===== R2 =====  (ether4 = .146, ether5 = .154)
# ===== R3 =====  (ether4 = .147, ether5 = .155)
```

R2 dan R3 pakai perintah sama dengan IP masing-masing. Untuk **backbone**, template **tanpa** `type=ptp` (default `broadcast`), karena segmen ini **shared** (banyak router satu segmen) → ada DR/BDR.

**Langkah 3: verifikasi neighbor & peran**

```
/routing ospf neighbor print
/routing ospf interface print
```

Yang dicatat: tiap router punya **4 entri neighbor** (2 router lain × 2 backbone), dan kolom state/peran tiap interface: **DR**, **BDR**, atau **DROther**.

Aturan pemilihan DR/BDR: **priority tertinggi** menang (default 128 sama), seri → **router-id tertinggi**. Jadi kalau semua router naik bersamaan, perkiraannya **R3 (3.3.3.3) = DR, R2 (2.2.2.2) = BDR, R1 = DROther**.

> **Hati-hati:** pemilihan DR **tidak preemptive** dan ada *wait timer*. Kalau routernya dikonfigurasi satu per satu dengan jeda lama, router yang naik duluan bisa jadi DR walau router-id-nya kecil. Jadi **catat yang benar-benar tampil** di Backbone A dan B (boleh beda antara A dan B).
> Dengan 3 router, DROther (R1) tetap `Full` ke DR dan BDR (satu-satunya neighbor-nya).

**Langkah 4: PREDIKSI** jumlah gateway R1 menuju IF-106.

```
# ===== R1 =====
/ip route print detail where dst-address=10.139.20.0/26
```

Jawaban dan alasan: **2 gateway**. R1 menjangkau R3 lewat Backbone A (`.147`) **dan** Backbone B (`.155`) dengan cost sama (masing-masing 1 + LAN R3 = 2). Jalur lewat R2 lebih mahal (lebih banyak hop). Dua jalur cost sama = **ECMP (Equal-Cost Multi-Path)**, traffic dibagi ke kedua gateway. Di output akan terlihat gateway `10.139.20.147` dan `10.139.20.155`.

**Langkah 5: failure pada backbone**

```bash
ping -O 10.139.20.10      # dari Laterano, biarkan jalan
```
Di GNS3: **Suspend** link antara **R3 `ether4` dan SW-BackboneA**. Amati di R1:

```
/ip route print detail where dst-address=10.139.20.0/26
```

Yang diamati:
- Gateway R1 berubah dari **2 jadi 1** (`.147` hilang, tersisa `.155` via Backbone B).
- **Durasi downtime tergantung flow yang kamu uji.** ECMP membagi per *flow* (pasangan IP), jadi ping Laterano→Kazdel biasanya memakai **satu** gateway saja. Kalau flow itu kebetulan lewat Backbone A → putus sampai OSPF mendeteksi (≈ dead interval 40 detik; kalau R3 adalah DR, bisa ada re-election tambahan). Kalau flow lewat Backbone B → **tidak putus sama sekali**. Kalau tidak ada downtime, **coba suspend link Backbone B** untuk melihat kasus lainnya, dan catat keduanya.
- Setelah **Resume**, gateway kembali 2.

**Langkah 6: DR priority**

```
# ===== R1 =====
/routing ospf interface-template set [find where comment="backbone ganda"] priority=0
/interface disable ether4,ether5
/interface enable ether4,ether5
/routing ospf interface print
```

Penjelasan: `priority=0` artinya router itu **tidak boleh jadi DR maupun BDR**. Disable/enable interface memicu **re-election** (pemilihan DR tidak berjalan ulang sendiri). Yang diamati: peran R1 menjadi **DROther** di kedua backbone. Kalau sebelumnya R1 sudah DROther, bandingkan dengan screenshot langkah 3 dan jelaskan bahwa perannya tetap, tapi sekarang **dijamin** tidak akan pernah jadi DR/BDR.

### 6.2 Problem D.2: Tambah Ruang Baca lewat R4

**Langkah 1: hitung di kertas (static routing + backup lewat backbone kedua).**

Ruang Baca `10.139.20.192/26` perlu dijangkau semua ruang. Dengan static + backup:

| Router | Route yang ditambah | Jumlah |
|---|---|---|
| R1, R2, R3 (masing-masing) | `.192/26` via R4 Backbone A (utama) + via R4 Backbone B (backup) | 2 × 3 = **6** |
| R4 | ke IF-106, IF-105, Ruang Dosen (masing-masing utama + backup) | 3 × 2 = **6** |
| **Total** | pada **4 router** | **12** |

Dengan OSPF: **0 route** di R1–R3; hanya **R4** yang dikonfigurasi. Itulah yang mau dibuktikan.

**Langkah 2: tambah R4** (CHR, **5 adapter**), **Switch4** dan **Higashi**. Link:

| Dari | Ke |
|---|---|
| Higashi `eth0` | Switch4 `e0` |
| Switch4 `e1` | R4 `ether1` |
| R4 `ether4` | SW-BackboneA `e3` |
| R4 `ether5` | SW-BackboneB `e3` |

Konfigurasi:

```
# ===== R4 =====
/ip dhcp-client remove [find]
/system identity set name=R4
/ip address
add address=10.139.20.193/26 interface=ether1 comment="LAN Ruang Baca"
add address=10.139.20.148/29 interface=ether4 comment="Backbone A"
add address=10.139.20.156/29 interface=ether5 comment="Backbone B"
/routing ospf instance add name=ospf1 version=2 router-id=4.4.4.4
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template
add area=backbone interfaces=ether4,ether5 comment="backbone ganda"
add area=backbone interfaces=ether1 passive comment="LAN"

# ===== Higashi =====
ip addr add 10.139.20.200/26 dev eth0
ip route add default via 10.139.20.193
```

> Walau router-id R4 (`4.4.4.4`) paling tinggi, R4 **tidak merebut DR**: pemilihan DR tidak preemptive, jadi DR/BDR yang sudah ada tetap. Dan R4 bisa sekadar `2-Way` ke R1 (sesama DROther), itu normal.

**Langkah 3: verifikasi TANPA menyentuh R1**

```
# ===== R1 =====   (hanya perintah baca!)
/ip route print where dst-address=10.139.20.192/26
```
```bash
# dari Laterano
mtr -r -c 5 10.139.20.200
```

Hasil: R1 **sudah punya** route `.192/26` berflag `o` (dua gateway: R4 `.148` & `.156`, ECMP) tanpa satu perintah pun di R1. Path: R1 `.129` → R4 (`.148` atau `.156`) → Higashi `.200` = **3 hop**. Higashi juga bisa dicoba ping ke Kazdel, Yan, dan Lungmen.

**Langkah 4: simpan konfigurasi akhir**

```
/export file=case4-R1        # R2, R3, R4: case4-R2, case4-R3, case4-R4
/file print                  # bukti file tersimpan
```

Untuk mengeluarkan file dari GNS3 (modul 0.5.3 & 0.5.4): lewat **WinBox → menu Files → Download**, atau **SFTP** (`get case4-R1.rsc`). Kalau topologimu tidak punya jalur ke luar, **cadangan bukti**: jalankan `/export` (tanpa `file=`) dan **screenshot isi terminal**.

---

## 7. Bagian E: Tugas Mandiri (Kertas)

Alokasi `10.139.50.0/24` (ganti dengan prefix-mu). Lab Jaringan (±60), Lab Sistem Cerdas (±25), Ruang Server (±6), chain **RA–RB–RC**, 2 link ptp.

> Asumsi: **RA = Lab Jaringan, RB = Lab Sistem Cerdas, RC = Ruang Server** (sesuai urutan di soal). Kalau asistenmu menentukan lain, tukar saja bagiannya.

### 7.1 Soal 1: VLSM (5 segmen)

Tambah 1 IP untuk gateway router (host + 1):

| Segmen | Butuh | Prefix |
|---|---|---|
| Lab Jaringan | 60 + 1 = 61 | /26 (62) |
| Lab Sistem Cerdas | 25 + 1 = 26 | /27 (30) |
| Ruang Server | 6 + 1 = 7 | /28 (14) |
| Link RA–RB | 2 | /30 |
| Link RB–RC | 2 | /30 |

> Ruang Server: 6 host + 1 gateway = 7 > 6 (batas /29) → **/28**. (Kalau soalmu menganggap 6 *sudah termasuk* router, boleh /29; tulis asumsimu.)

Alokasi berurutan:

| No | Parameter | Lab Jaringan (±60) | Lab Sistem Cerdas (±25) | Ruang Server (±6) | Link RA–RB | Link RB–RC |
|---|---|---|---|---|---|---|
| 1 | Prefix | /26 | /27 | /28 | /30 | /30 |
| 2 | Network Address | 10.139.50.0 | 10.139.50.64 | 10.139.50.96 | 10.139.50.112 | 10.139.50.116 |
| 3 | Broadcast Address | 10.139.50.63 | 10.139.50.95 | 10.139.50.111 | 10.139.50.115 | 10.139.50.119 |
| 4 | Rentang Host Valid | .1 – .62 | .65 – .94 | .97 – .110 | .113 – .114 | .117 – .118 |

### 7.2 Soal 2: Rencana alamat interface

(host pertama LAN ke ether1; host pertama link ke router bernomor lebih kecil)

| Router | ether1 (LAN) | ether2 | ether3 |
|---|---|---|---|
| RA | 10.139.50.1/26 | 10.139.50.113/30 (ke RB) | - |
| RB | 10.139.50.65/27 | 10.139.50.114/30 (ke RA) | 10.139.50.117/30 (ke RC) |
| RC | 10.139.50.97/28 | 10.139.50.118/30 (ke RB) | - |

### 7.3 Soal 3: Routing table + tandai summarization

| Router | Destination | Next Hop | Interface | Metode |
|---|---|---|---|---|
| RA | Lab Jaringan 10.139.50.0/26 | - | ether1 | Terhubung Langsung |
| RA | Link RA–RB 10.139.50.112/30 | - | ether2 | Terhubung Langsung |
| RA | Lab Sistem Cerdas 10.139.50.64/27 | 10.139.50.114 | ether2 | Static |
| RA | Ruang Server 10.139.50.96/28 | 10.139.50.114 | ether2 | Static |
| RB | Lab Sistem Cerdas 10.139.50.64/27 | - | ether1 | Terhubung Langsung |
| RB | Link RA–RB 10.139.50.112/30 | - | ether2 | Terhubung Langsung |
| RB | Link RB–RC 10.139.50.116/30 | - | ether3 | Terhubung Langsung |
| RB | Lab Jaringan 10.139.50.0/26 | 10.139.50.113 | ether2 | Static |
| RB | Ruang Server 10.139.50.96/28 | 10.139.50.118 | ether3 | Static |
| RC | Ruang Server 10.139.50.96/28 | - | ether1 | Terhubung Langsung |
| RC | Link RB–RC 10.139.50.116/30 | - | ether2 | Terhubung Langsung |
| RC | Lab Jaringan 10.139.50.0/26 | 10.139.50.117 | ether2 | Static |
| RC | Lab Sistem Cerdas 10.139.50.64/27 | 10.139.50.117 | ether2 | Static |

**Yang bisa di-summarize:**
- **RA:** `.64/27` + `.96/28` (next hop sama `.114`, berurutan 64–111) → **`10.139.50.64/26`** (64–127).
- **RC:** `.0/26` + `.64/27` (next hop sama `.117`, 0–95) → **`10.139.50.0/25`** (0–127). Blok ini juga mencakup LAN RC sendiri, tapi route connected `/28` lebih spesifik sehingga aman (LPM).
- **RB:** tidak bisa (next hop beda).

### 7.4 Soal 4: Link RC–RA sebagai backup + alasan dynamic routing

Dengan n = 3 dan tiap tujuan butuh 1 backup, tambahan **2 backup route per router × 3 router = 6 static route tambahan** (total 12, sesuai tabel A.3 untuk n = 3). Plus alamat link baru (`.120/30`) dan atur `distance` backup agar tidak bentrok dengan route utama.

Alasan memilih **dynamic routing (OSPF)**:
1. Route (utama + backup) dihitung otomatis dari topologi, tidak ditulis satu per satu.
2. Failover otomatis saat link putus; static butuh `check-gateway` per route dan tetap lambat.
3. Tambah link/ruang cukup konfigurasi di router yang berubah (lihat D.2), sedangkan static menyentuh semua router (A.3).
4. Mengurangi risiko salah ketik, routing loop, dan route yang tidak simetris.

---

## 8. Troubleshooting Sistematis

Soal meminta *mendiagnosis secara sistematis*. Pegang urutan ini, **dari dekat ke jauh, lalu cek arah pulang**.

### 8.1 Static routing / konektivitas

| Langkah | Perintah | Kalau gagal → kemungkinan |
|---|---|---|
| 1. Interface hidup? | `/interface print` | Kabel salah port, router belum start (flag `R` tidak ada) |
| 2. IP benar? | `/ip address print`, `ip a` di host | Salah ketik prefix, salah mask, IP bawaan DHCP masih ada |
| 3. Host ping gateway-nya | `ping <gateway>` | Gateway host salah / mask beda subnet |
| 4. Router ping next hop | `/ping <next hop>` | Link/IP link salah |
| 5. Cek route tujuan | `/ip route print` | Route belum ada (`Destination Net Unreachable`) |
| 6. Cek next hop route | kolom GATEWAY | Gateway tidak di subnet connected (route `Inactive`) |
| 7. **Cek route balik** | `/ip route print` di router tujuan | Return route kurang → ping gagal tanpa pesan |
| 8. Lihat path | `mtr` / `/tool traceroute` | Hop terakhir yang muncul = router **sebelum** titik putus |
| 9. Loop? | `Time to live exceeded`, hop berulang | Gateway saling menunjuk |
| 10. Log | `/log print` | Riwayat perubahan konfigurasi |

Pola gejala:
- **Timeout tanpa pesan** → route balik hilang atau ada blackhole.
- **`Destination Net Unreachable` dari router X** → router X tidak punya route ke tujuan.
- **`Time to live exceeded`** → routing loop.
- **Satu arah jalan, arah sebaliknya tidak** → asymmetric/return route.

### 8.2 OSPF

| Gejala | Cek | Penyebab umum |
|---|---|---|
| Neighbor tidak muncul | `/routing ospf interface print`, `neighbor print` | Interface belum masuk template; interface `disabled`; kabel salah |
| Neighbor macet di `Init` / `2-Way` / `ExStart` | konfigurasi kedua sisi | Beda **area-id**, beda **hello/dead interval**, beda **type** (ptp vs broadcast), **mask** beda, IP link salah |
| Neighbor ada, route tidak muncul | `/ip route print where ospf` | Salah `passive` di interface link, subnet LAN tidak masuk template |
| Dua router sama persis | `router-id` | **router-id duplikat** |
| Tidak pulih setelah link putus | `neighbor print` | Belum lewat dead interval (40 detik) |
| DR/BDR tidak sesuai harapan | `interface print` | Pemilihan tidak preemptive → disable/enable interface agar re-election |

---

## 9. Checklist Screenshot & Hari-H

### 9.1 Sebelum praktikum (malam ini)

- [ ] Hitung ulang Bagian A **di kertas tanpa lihat** jawaban di sini, lalu cocokkan.
- [ ] Hafal pola perintah: `/ip address add address=X/NN interface=etherN`, `/ip route add dst-address=X/NN gateway=Y`.
- [ ] Pahami **kenapa** (bukan cuma copas): return route, LPM, TTL tiap hop, `passive`, `ptp` vs broadcast, DR/BDR.
- [ ] Coba buat topologi B di GNS3 sekali sebagai latihan (adapter 5!).
- [ ] Pastikan Wireshark terpasang dan tombol *Start capture* di GNS3 bisa membuka Wireshark.
- [ ] Pastikan appliance MikroTik CHR dan netics-pc(-desktop) sudah terinstal.
- [ ] Cari tahu **IP Prefix** yang kamu pakai.

### 9.2 Urutan kerja hari-H (ringkas)

1. **A** (kertas) → isi tabel A.1, A.2, A.3, cocokkan dengan Tabel Alokasi Acuan.
2. **B.1**: topologi → adapter 5 → link → login → `dhcp-client remove` + `identity` → IP router → IP host → `route print` → **prediksi** → ping.
3. **B.2**: route R1+R2 → **prediksi** → ping/traceroute R3 → route R3 → mtr → Wireshark TTL/proto/checksum.
4. **B.3**: summarization → LPM (**prediksi**, blackhole /32, hapus) → loop (**prediksi**, ubah gateway R2, lihat capture, kembalikan).
5. **C.1**: link R3–R1 → IP → direct route + distance → mtr → asymmetric → `ping -O` + Suspend → status interface → check-gateway.
6. **C.2**: hapus static → OSPF (instance → area → template) → neighbor/interface/route → Suspend → Resume.
7. **D.1**: switch backbone → link ether4/ether5 → disable ether2/3 → IP + template → DR/BDR → ECMP → Suspend backbone → priority 0.
8. **D.2**: hitung static di kertas → R4 + Higashi → verifikasi R1 tanpa disentuh → `/export`.
9. **E**: kertas.

### 9.3 Daftar semua screenshot yang diminta soal

| Bagian | Yang discreenshot |
|---|---|
| B.1 | `/ip address print` (R1,R2,R3), `/ip route print` awal, ping ke `.162` |
| B.2 | Prediksi tertulis, ping+traceroute R3, `/ip route print` lengkap, 3× mtr, TTL di 3 capture, Protocol+Checksum di 3 capture |
| B.3 | Route R1 setelah summary, ping setelah summary, uji LPM (3 ping), pesan loop, TTL berulang di capture R1–R2, ping setelah dikembalikan |
| C.1 | mtr 3 hop vs B.2, mtr asimetris (2 arah), ping saat Suspend + `interface print` + route print, hasil check-gateway + durasi |
| C.2 | neighbor/interface/route OSPF, route R1 + neighbor saat Suspend + durasi, kondisi setelah Resume |
| D.1 | DR/BDR di Backbone A dan B, route detail 2 gateway, perubahan gateway + durasi saat Suspend, peran R1 setelah priority 0 |
| D.2 | Perhitungan kertas, route `.192/26` di R1, mtr ke Higashi, file `case4-R*` |
| E | Tabel VLSM, alokasi interface, routing table, jawaban soal 4 |

> Tips terakhir: **jangan lupa kembalikan konfigurasi** setelah setiap manipulasi (blackhole dihapus, gateway R2 dikembalikan, link di-Resume). Kalau lupa, bagian berikutnya akan memberi hasil yang membingungkan.
>
> Semangat! Kamu pasti bisa. 🔥
