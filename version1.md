# Modul 4 Jaringan Komputer — Panduan Praktikum Lengkap

> Panduan belajar dan checklist pengerjaan berdasarkan file **Pra-Praktikum Modul 4** dan **README Modul 4 Jaringan Komputer**.
>
> **Prefix contoh yang dipakai:** `10.126`. Jika prefix pribadi/kelompokmu berbeda, ganti `10.126` di seluruh konfigurasi. Jangan mengubah angka host terakhir dari tabel alokasi acuan tanpa alasan.

## 1. Gambaran besar: sebenarnya praktikum ini mau membuktikan apa?

Modul 4 membahas bagaimana beberapa subnet/ruang yang berbeda dapat saling berkomunikasi melalui router, lalu bagaimana jaringan tetap berjalan ketika jalur utama putus dan bagaimana jaringan diperluas.

Urutan logikanya:

1. **Bagian A — rencanakan dulu:** hitung subnet dengan VLSM, tentukan IP setiap interface, lalu gambar routing table secara manual.
2. **Bagian B — static routing:** bangun tiga router berantai. Buktikan bahwa router hanya otomatis mengenal jaringan yang menempel langsung; jaringan lain harus diberi route. Pelajari rute pergi-pulang, summarization, longest prefix match, dan routing loop.
3. **Bagian C — ring dan OSPF:** tambahkan link langsung R1–R3. Bandingkan static route saat link gagal, lalu ganti static routing dengan OSPF dan amati pemulihan otomatis.
4. **Bagian D — dua backbone dan router baru:** ganti link khusus antar-router dengan dua jaringan backbone bersama. Amati DR/BDR, ECMP, dan buktikan router baru/R4 dapat mengiklankan LAN baru tanpa mengubah konfigurasi R1–R3.

**Inti yang perlu kamu pahami:** setiap router mengambil keputusan berdasarkan routing table miliknya sendiri. Ping berhasil hanya jika jalur menuju tujuan dan jalur balasan sama-sama tersedia.

---

## 2. Kenali node dan port sebelum menyentuh konfigurasi

### Pemetaan node

| Node | Peran |
|---|---|
| R1 | Router Ruang Dosen |
| R2 | Router IF-105 |
| R3 | Router IF-106 |
| R4 | Router Ruang Baca (hanya Bagian D) |
| Lungmen | Host uji Ruang Dosen |
| Laterano | Host pengamatan Ruang Dosen; dipakai untuk ping dan `mtr` |
| Yan | Host IF-105 |
| Kazdel | Host IF-106 |
| Higashi | Host Ruang Baca (Bagian D) |

### Fungsi port yang baku

| Interface | Fungsi |
|---|---|
| `ether1` | LAN ruang; IP router di sini menjadi default gateway host |
| `ether2` | Link point-to-point ke router tetangga pertama |
| `ether3` | Link point-to-point ke router tetangga kedua |
| `ether4` | Backbone A |
| `ether5` | Backbone B |

Jangan menukar fungsi port. Di Bagian B/C, router dihubungkan dengan link point-to-point. Di Bagian D, link itu dinonaktifkan dan diganti dua backbone melalui switch.

### Perintah pemeriksaan yang paling sering dipakai

```routeros
/ip address print
/ip route print
/ip route print detail
/interface print
/routing ospf neighbor print
/routing ospf interface print
/routing rip neighbor print
/log print
```

Di terminal host Linux:

```bash
ip addr
ip route
ping -c 3 ALAMAT_TUJUAN
mtr -r -c 5 ALAMAT_TUJUAN
traceroute ALAMAT_TUJUAN
```

`/ip ...` adalah perintah RouterOS pada router MikroTik. `ip ...`, `ping`, `mtr`, dan `traceroute` adalah perintah Linux pada PC.

---

## 3. Konsep wajib sebelum praktikum

### 3.1 Subnet, prefix, dan jumlah host

IPv4 memiliki 32 bit. Prefix `/26` berarti 26 bit untuk network dan 6 bit untuk host.

Jumlah alamat dalam subnet = `2^(32-prefix)`.
Jumlah host usable pada subnet biasa = `2^(32-prefix) - 2` (network dan broadcast tidak dipakai host).

| Prefix | Total alamat | Host usable | Ukuran blok oktet terakhir |
|---|---:|---:|---:|
| `/30` | 4 | 2 | 4 |
| `/29` | 8 | 6 | 8 |
| `/28` | 16 | 14 | 16 |
| `/27` | 32 | 30 | 32 |
| `/26` | 64 | 62 | 64 |
| `/25` | 128 | 126 | 128 |
| `/24` | 256 | 254 | 256 |

Istilah:
- **Network address:** identitas subnet, tidak diberikan ke perangkat.
- **Broadcast address:** alamat broadcast subnet, tidak diberikan ke perangkat.
- **Host range:** alamat yang boleh dipakai perangkat.
- **Gateway:** IP interface router yang berada dalam LAN host.
- **Next hop:** IP router tetangga yang menjadi tujuan penerusan paket berikutnya.
- **Directly connected:** subnet yang langsung menempel pada interface router; otomatis masuk routing table.

### 3.2 VLSM

VLSM adalah membagi satu blok induk menjadi subnet dengan ukuran berbeda sesuai kebutuhan host. Kerjakan dari kebutuhan host terbesar ke terkecil supaya blok besar mendapat tempat lebih dulu.

Cara cepat:
1. Urutkan kebutuhan host dari terbesar.
2. Cari prefix terkecil yang masih cukup.
3. Alokasikan blok dari alamat awal secara berurutan.
4. Pastikan network address sesuai kelipatan ukuran blok dan tidak overlap.
5. Sisakan blok yang sesuai untuk subnet cadangan.

### 3.3 Static routing dan multi-hop

Static route adalah rute yang dimasukkan manual oleh administrator. Setiap router hanya meneruskan paket ke **next hop** sesuai routing table-nya.

Contoh alur Laterano → Yan pada chain:
`Laterano → R1 → R2 → Yan`.

Agar ping sukses, balasannya juga harus punya jalur:
`Yan → R2 → R1 → Laterano`.

Kalau route pergi ada tetapi route pulang tidak ada, request mungkin sampai ke tujuan tetapi reply tidak dapat kembali. Karena itu, jangan hanya memeriksa router pengirim.

### 3.4 Route summarization

Menggabungkan beberapa subnet berurutan menjadi satu route yang lebih besar/prefix lebih pendek. Syarat penting:
- subnet berurutan dan sejajar pada batas blok;
- seluruh subnet yang digabung menuju next hop yang sama.

Manfaat: routing table lebih ringkas. Risiko: summary dapat mencakup alamat yang belum digunakan.

### 3.5 Longest Prefix Match (LPM)

Jika satu IP tujuan cocok dengan beberapa route, router memilih prefix paling panjang/paling spesifik.

Urutan contoh: `/32` lebih spesifik daripada `/26`, dan `/26` lebih spesifik daripada `/25`.
Host route `/32` hanya berlaku untuk satu alamat IP.

### 3.6 Routing loop dan TTL

Routing loop terjadi ketika router saling mengirim paket untuk tujuan yang sama karena route yang keliru. TTL berkurang satu setiap melewati router. Saat TTL habis, paket dibuang dan pengirim menerima ICMP `Time-to-live exceeded`.

### 3.7 Dynamic routing: RIP dan OSPF

Dynamic routing membuat router bertukar informasi jaringan sehingga route dapat dipelajari otomatis.

- **RIP:** distance-vector; memilih jalur berdasarkan hop count. Lebih sederhana, tetapi konvergensinya relatif lambat.
- **OSPF:** link-state; router bertukar LSA, membangun LSDB, lalu menghitung jalur dengan algoritma SPF/Dijkstra berdasarkan cost. Umumnya lebih cepat merespons perubahan.
- **Convergence:** waktu sampai router-router kembali memiliki informasi route yang konsisten setelah perubahan topologi.
- **Failover:** perpindahan traffic dari jalur utama ke jalur alternatif ketika jalur utama gagal.
- **Administrative distance (distance di RouterOS):** dipakai untuk memilih di antara route dari sumber berbeda/route dengan tujuan setara. Nilai lebih kecil lebih diutamakan.
- **ECMP:** beberapa jalur dengan cost sama dapat dipasang sebagai beberapa next hop aktif.
- **DR/BDR:** pada segmen OSPF broadcast, Designated Router dan Backup Designated Router membantu mengatur pertukaran informasi OSPF. Pemilihannya dipengaruhi priority dan router ID; priority 0 membuat router tidak eligible menjadi DR/BDR.

---

# BAGIAN A — PERENCANAAN DI KERTAS

Bagian A bukan konfigurasi GNS3. Hasilnya menjadi acuan alamat untuk Bagian B, C, dan D.

## A.1 Hitung VLSM untuk seluruh segmen

Soal memakai alokasi `10.126.20.0/24` (contoh prefix `10.126`) dan kebutuhan:

| Segmen | Kebutuhan |
|---|---:|
| IF-106 | ±50 host |
| IF-105 | ±40 host |
| Ruang Dosen | ±10 host |
| Backbone A | hingga 6 router |
| Backbone B | hingga 6 router |
| Link R1–R2 | 2 router |
| Link R2–R3 | 2 router |
| Link R3–R1 | 2 router |
| Ruang Baca cadangan | ±35 host |

### Hasil alokasi acuan

| Segmen | Network/prefix | Host usable | Broadcast |
|---|---|---|---|
| IF-106 | `10.126.20.0/26` | `.1 – .62` | `.63` |
| IF-105 | `10.126.20.64/26` | `.65 – .126` | `.127` |
| Ruang Dosen | `10.126.20.128/28` | `.129 – .142` | `.143` |
| Backbone A | `10.126.20.144/29` | `.145 – .150` | `.151` |
| Backbone B | `10.126.20.152/29` | `.153 – .158` | `.159` |
| Link R1–R2 | `10.126.20.160/30` | `.161 – .162` | `.163` |
| Link R2–R3 | `10.126.20.164/30` | `.165 – .166` | `.167` |
| Link R3–R1 | `10.126.20.168/30` | `.169 – .170` | `.171` |
| Ruang Baca | `10.126.20.192/26` | `.193 – .254` | `.255` |

Catatan: `.172–.191` tidak dipakai di tabel acuan; Ruang Baca dimulai di `.192` karena network `/26` harus sejajar pada kelipatan 64.

**Kenapa prefix-nya begitu?**
- 50 host perlu minimal 62 host usable → `/26`.
- 40 host perlu minimal 62 host usable → `/26`.
- 10 host perlu minimal 14 host usable → `/28`.
- Backbone maksimal 6 router → `/29` menyediakan 6 host usable.
- Link point-to-point perlu 2 alamat → `/30`.
- Ruang Baca 35 host → `/26`.

## A.2 Rencana IP interface

Aturan: `ether1` router memakai host pertama LAN. Pada link antar-router, router bernomor lebih kecil memakai IP pertama.

| Router | ether1 (LAN) | ether2 | ether3 |
|---|---|---|---|
| R1 | `10.126.20.129/28` | `10.126.20.161/30` ke R2 | belum dipakai di chain |
| R2 | `10.126.20.65/26` | `10.126.20.162/30` ke R1 | `10.126.20.165/30` ke R3 |
| R3 | `10.126.20.1/26` | `10.126.20.166/30` ke R2 | belum dipakai di chain |

Saat ring dibentuk, tambahkan:
- R1 `ether3 = 10.126.20.169/30`
- R3 `ether3 = 10.126.20.170/30`

Backbone:
- Backbone A: R1 `.145`, R2 `.146`, R3 `.147`, R4 `.148`, semuanya `/29`.
- Backbone B: R1 `.153`, R2 `.154`, R3 `.155`, R4 `.156`, semuanya `/29`.

Ruang Baca:
- R4 `ether1 = 10.126.20.193/26`
- Higashi `eth0 = 10.126.20.200/26`, gateway `10.126.20.193`.

## A.2 Routing table chain — rencanakan sebelum konfigurasi

**R1**
- Connected: Ruang Dosen `10.126.20.128/28`, Link R1–R2 `10.126.20.160/30`.
- Static ke IF-105 `10.126.20.64/26` via `10.126.20.162`.
- Static ke IF-106 `10.126.20.0/26` via `10.126.20.162`.

**R2**
- Connected: IF-105 `10.126.20.64/26`, Link R1–R2 `10.126.20.160/30`, Link R2–R3 `10.126.20.164/30`.
- Static ke Ruang Dosen `10.126.20.128/28` via `10.126.20.161`.
- Static ke IF-106 `10.126.20.0/26` via `10.126.20.166`.

**R3**
- Connected: IF-106 `10.126.20.0/26`, Link R2–R3 `10.126.20.164/30`.
- Static ke Ruang Dosen `10.126.20.128/28` via `10.126.20.165`.
- Static ke IF-105 `10.126.20.64/26` via `10.126.20.165`.

R1 dapat menggabungkan dua route IF-105 dan IF-106 menjadi `10.126.20.0/25` via R2 karena kedua subnet berurutan dan next hop-nya sama.

## A.3 Beban static route

Jika setiap router memiliki satu LAN dan semua router harus dapat menjangkau LAN router lain:
- Static route per router = `n - 1`.
- Total static route tanpa backup = `n(n - 1)`.
- Jika setiap tujuan membutuhkan satu route alternatif/backup, perkiraan total entri route = `2n(n - 1)` (dua pilihan jalur untuk tiap tujuan pada tiap router).
- Saat menambah satu ruang, router lama perlu diberi route menuju LAN baru; router baru juga perlu route menuju LAN lama. Dengan asumsi seluruh router menyimpan route spesifik, seluruh `n+1` router perlu disentuh.

| Jumlah router | Route/router | Total tanpa backup | Total dengan satu backup/tujuan |
|---:|---:|---:|---:|
| 3 | 2 | 6 | 12 |
| 4 | 3 | 12 | 24 |
| 10 | 9 | 90 | 180 |
| n | n−1 | n(n−1) | 2n(n−1) |

---

# BAGIAN B — STATIC ROUTING CHAIN

## B.0 Topologi yang dibangun

`Laterano/Lungmen — switch Ruang Dosen — R1 — R2 — R3 — switch IF-106 — Kazdel`

R2 juga terhubung ke switch IF-105 dan Yan. Link:
- R1 `ether2` ↔ R2 `ether2`
- R2 `ether3` ↔ R3 `ether2`
- ether1 tiap router ↔ switch LAN ruangnya.

## B.1 Membangun chain dan memeriksa connected route

### Tujuan

Membuktikan router yang baru dikonfigurasi hanya mengenal subnet yang terhubung langsung. Belum ada static route, jadi komunikasi lintas subnet belum berjalan.

### 1. Persiapan router

Jalankan pada masing-masing router; sesuaikan identity:

```routeros
/ip dhcp-client remove [find]
/system identity set name=R1
```

Ulangi di R2 dan R3 dengan nama masing-masing. Pastikan router memakai 5 adapter sesuai kebutuhan modul.

### 2. Beri IP pada interface router

**R1**
```routeros
/ip address
add address=10.126.20.129/28 interface=ether1 comment="LAN Ruang Dosen"
add address=10.126.20.161/30 interface=ether2 comment="link ke R2"
```

**R2**
```routeros
/ip address
add address=10.126.20.65/26 interface=ether1 comment="LAN IF-105"
add address=10.126.20.162/30 interface=ether2 comment="link ke R1"
add address=10.126.20.165/30 interface=ether3 comment="link ke R3"
```

**R3**
```routeros
/ip address
add address=10.126.20.1/26 interface=ether1 comment="LAN IF-106"
add address=10.126.20.166/30 interface=ether2 comment="link ke R2"
```

### 3. Atur IP host dan default gateway

**Lungmen**
```bash
ip addr add 10.126.20.130/28 dev eth0
ip route add default via 10.126.20.129
```

**Laterano**
```bash
ip addr add 10.126.20.131/28 dev eth0
ip route add default via 10.126.20.129
```

**Yan**
```bash
ip addr add 10.126.20.70/26 dev eth0
ip route add default via 10.126.20.65
```

**Kazdel**
```bash
ip addr add 10.126.20.10/26 dev eth0
ip route add default via 10.126.20.1
```

### 4. Buktikan connected route

Pada setiap router:

```routeros
/ip address print
/ip route print
```

Cari route ber-flag connected (`DAc`/connected, tampilan flag dapat bervariasi). Cocokkan dengan LAN dan link yang benar-benar terpasang.

### 5. Prediksi dan uji

Sebelum ping, tulis: **Laterano belum dapat menjangkau Yan**, karena R1 belum punya route menuju IF-105 dan R2 belum punya route kembali menuju Ruang Dosen.

```bash
# Dari Laterano: uji link R1-R2 dan Yan
ping -c 3 10.126.20.162
ping -c 3 10.126.20.70
```

Ping ke `.162` menguji alamat tetangga pada link langsung; ping ke Yan menguji komunikasi lintas LAN yang memerlukan routing.

### Bukti/screenshot B.1

- `/ip address print` R1, R2, R3.
- `/ip route print` awal masing-masing router.
- Ping Laterano ke `10.126.20.162`.
- Ping Laterano ke Yan (`10.126.20.70`) yang belum berhasil.

## B.2 Static route multi-hop, return path, dan datagram

### Tujuan

Membuat tiga LAN saling terhubung. Membuktikan route harus ada pada semua router yang dilewati, termasuk jalur balasan. Kemudian mengamati TTL dan checksum paket di Wireshark.

### 1. Pasang static route di R1 dan R2

**R1**
```routeros
/ip route
add dst-address=10.126.20.64/26 gateway=10.126.20.162 comment="IF-105 via R2"
add dst-address=10.126.20.0/26 gateway=10.126.20.162 comment="IF-106 via R2"
```

**R2**
```routeros
/ip route
add dst-address=10.126.20.128/28 gateway=10.126.20.161 comment="Ruang Dosen via R1"
add dst-address=10.126.20.0/26 gateway=10.126.20.166 comment="IF-106 via R3"
```

### 2. Prediksi sebelum ping

R3 belum punya route ke Ruang Dosen. Request dari Laterano dapat diteruskan R1 → R2 → R3, tetapi reply dari Kazdel tidak tahu cara kembali ke Ruang Dosen. Maka ping belum berhasil; masalahnya ada pada route balik di R3.

Uji:

```bash
# Laterano
ping -c 3 10.126.20.10
```

Di R3:
```routeros
/tool traceroute 10.126.20.131
```

### 3. Lengkapi route di R3

```routeros
/ip route
add dst-address=10.126.20.128/28 gateway=10.126.20.165 comment="Ruang Dosen via R2"
add dst-address=10.126.20.64/26 gateway=10.126.20.165 comment="IF-105 via R2"
```

Sekarang R3 tahu jalur pulang menuju Ruang Dosen dan IF-105.

### 4. Verifikasi jalur dari dua arah

**Laterano**
```bash
mtr -r -c 5 10.126.20.70
mtr -r -c 5 10.126.20.10
```

**Kazdel**
```bash
mtr -r -c 5 10.126.20.131
```

Ekspektasi jalur:
- Ruang Dosen → IF-105: R1 → R2.
- Ruang Dosen → IF-106: R1 → R2 → R3.
- IF-106 → Ruang Dosen: R3 → R2 → R1.

### 5. Capture ICMP dengan Wireshark

1. Klik kanan link R1–R2 → **Start capture**.
2. Lakukan hal yang sama pada link R2–R3 dan link switch Ruang Dosen–R1.
3. Di Wireshark gunakan filter `icmp`.
4. Dari Laterano:
   ```bash
   ping -c 3 10.126.20.10
   ```
5. Cari Echo Request yang sama pada ketiga capture. Buka `Internet Protocol Version 4`.

Field yang diamati:
- `ip.ttl`: TTL berkurang satu pada setiap router yang dilewati.
- `ip.proto`: ICMP bernilai `1`.
- `ip.checksum`: checksum header IPv4 dihitung ulang ketika field TTL berubah.

**Bukti/screenshot B.2**
- Prediksi sebelum R3 dikonfigurasi.
- Ping dan traceroute dari R3.
- `/ip route print` lengkap R1/R2/R3.
- Tiga hasil `mtr` sesuai arah.
- Capture Wireshark untuk LAN Ruang Dosen, link R1–R2, dan link R2–R3 yang memperlihatkan TTL, protocol, checksum.

## B.3 Manipulasi route

### (a) Route summarization

**Tujuan:** mengganti dua route R1 menuju IF-105 dan IF-106 menjadi satu route ringkasan karena keduanya berada di belakang R2.

Di R1:
```routeros
/ip route remove [find where comment~"via R2"]
/ip route add dst-address=10.126.20.0/25 gateway=10.126.20.162 comment="IF-105 dan IF-106 via R2 (summary)"
/ip route print
```

Uji dari Laterano:
```bash
ping -c 3 10.126.20.70
ping -c 3 10.126.20.10
```

Keduanya harus tetap dapat dijangkau. Summary `/25` mencakup `.0–.127`, yakni IF-106 dan IF-105.

### (b) Longest Prefix Match

**Tujuan:** membuktikan route paling spesifik menang.

Di R1:
```routeros
/ip route add dst-address=10.126.20.10/32 blackhole comment="uji LPM"
```

Prediksi:
- Kazdel `.10`: gagal karena cocok dengan route blackhole `/32`, yang lebih spesifik.
- R3 ether1 `.1`: tetap mengikuti route summary `/25` (tetapi perlu diingat `.1` adalah IP interface R3, sehingga respons ping bergantung pada kondisi router).
- Yan `.70`: tetap mengikuti summary `/25`, jadi seharusnya tetap terjangkau.

Uji:
```bash
ping -c 3 10.126.20.10
ping -c 3 10.126.20.1
ping -c 3 10.126.20.70
```

Hapus route eksperimen:
```routeros
/ip route remove [find where comment="uji LPM"]
```

### (c) Routing loop

**Tujuan:** sengaja membuat R2 mengirim tujuan IF-106 kembali ke R1, sementara R1 mengirimnya ke R2. Paket berputar dan TTL terus turun.

Di R2:
```routeros
/ip route set [find where comment="IF-106 via R3"] gateway=10.126.20.161
```

Dari Laterano:
```bash
ping -c 2 10.126.20.10
```

Di Wireshark capture link R1–R2, filter `icmp`. Amati Echo Request yang sama melintas berulang dengan TTL menurun sampai pesan `Time-to-live exceeded`.

**Pulihkan segera:**
```routeros
/ip route set [find where comment="IF-106 via R3"] gateway=10.126.20.166
```

Lalu uji ping Kazdel lagi.

**Bukti/screenshot B.3**
- Routing table R1 setelah summary.
- Ping Yan dan Kazdel setelah summary.
- Prediksi dan hasil ping `.10`, `.1`, `.70` saat route `/32` blackhole.
- Pesan saat loop dan TTL berulang di capture.
- Ping berhasil lagi setelah konfigurasi dipulihkan.

---

# BAGIAN C — REDUNDANSI DAN OSPF PADA TOPOLOGI RING

## C.1 Redundant link dengan static routing

### Tujuan

Menambah link langsung R1–R3. Jalur Ruang Dosen ↔ IF-106 menjadi lebih pendek. Static route dipakai untuk memilih jalur utama dan route lain sebagai cadangan. Uji apa yang terjadi saat link langsung putus.

### 1. Tambahkan link R1–R3

Sambungkan R1 `ether3` ↔ R3 `ether3`.

**R1**
```routeros
/ip address add address=10.126.20.169/30 interface=ether3 comment="link ke R3"
```

**R3**
```routeros
/ip address add address=10.126.20.170/30 interface=ether3 comment="link ke R1"
```

### 2. Tambahkan direct route

**R1:** route langsung `/26` ke IF-106 lebih spesifik daripada summary `/25`, sehingga dipilih untuk tujuan IF-106.
```routeros
/ip route add dst-address=10.126.20.0/26 gateway=10.126.20.170 comment="IF-106 langsung via R3"
```

**R3:** route langsung ke Ruang Dosen menjadi jalur utama. Route lama via R2 dibuat distance lebih besar sebagai backup.
```routeros
/ip route set [find where comment="Ruang Dosen via R2"] distance=10
/ip route add dst-address=10.126.20.128/28 gateway=10.126.20.169 comment="Ruang Dosen langsung via R1"
```

### 3. Verifikasi jalur lebih pendek

Dari Laterano:
```bash
mtr -r -c 5 10.126.20.10
```

Bandingkan dengan B.2. Sebelumnya ke IF-106 melewati R1–R2–R3; sekarang jalur langsung R1–R3.

### 4. Uji asymmetric path

Di R3, disable route langsung ke Ruang Dosen:
```routeros
/ip route disable [find where comment="Ruang Dosen langsung via R1"]
```

Jalankan:
```bash
# Laterano
mtr -r -c 5 10.126.20.10

# Kazdel
mtr -r -c 5 10.126.20.131
```

Penalaran: keputusan routing tiap arah dibuat terpisah oleh router asal. Satu arah bisa melalui direct link, sedangkan arah sebaliknya melalui R2. Ini disebut asymmetric path.

Aktifkan kembali:
```routeros
/ip route enable [find where comment="Ruang Dosen langsung via R1"]
```

### 5. Uji link failure

1. Dari Laterano jalankan ping berkelanjutan:
   ```bash
   ping -O 10.126.20.10
   ```
2. Saat ping berjalan, klik kanan link R1–R3 di GNS3 → **Suspend**.
3. Prediksi: static route yang mengarah ke link itu belum tentu otomatis pindah jika router tidak mendeteksi next hop gagal. Amati hasil aktual.
4. Di R1:
   ```routeros
   /interface print where name=ether3
   /ip route print where dst-address=10.126.20.0/26
   ```
5. Resume link R1–R3.
6. Aktifkan gateway checking pada kedua direct route:
   ```routeros
   # R1
   /ip route set [find where comment="IF-106 langsung via R3"] check-gateway=ping

   # R3
   /ip route set [find where comment="Ruang Dosen langsung via R1"] check-gateway=ping
   ```
7. Ulangi suspend sambil ping berjalan. Catat berapa lama ping terputus sebelum route cadangan digunakan. Resume setelah selesai.

**Bukti C.1**
- IP link baru.
- Routing table yang menunjukkan direct route dan backup distance.
- MTR jalur langsung.
- MTR dua arah saat asymmetric path.
- Ping berkelanjutan saat link suspend.
- Status ether3 dan route ketika link gagal.
- Waktu/durasi failover setelah `check-gateway=ping`.

## C.2 OSPF pada topologi ring

### Tujuan

Menghapus static route dan membiarkan OSPF mempelajari subnet serta memilih jalur secara otomatis. Kemudian ulangi link failure untuk mengamati convergence.

### 1. Hapus seluruh static route

Pada R1, R2, R3:
```routeros
/ip route remove [find where static]
/ip route print
```

Pastikan route connected masih ada. Jangan hapus alamat interface.

### 2. Konfigurasi OSPF di setiap router

OSPF dibuat berurutan:
1. **instance**: proses OSPF dan router ID.
2. **area**: area tempat router berpartisipasi.
3. **interface-template**: interface yang menjalankan OSPF.

**R1**
```routeros
/routing ospf instance add name=ospf1 version=2 router-id=1.1.1.1
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template
add area=backbone interfaces=ether2,ether3 type=ptp comment="link inter-router"
add area=backbone interfaces=ether1 passive comment="LAN"
```

**R2** — router ID `2.2.2.2`:
```routeros
/routing ospf instance add name=ospf1 version=2 router-id=2.2.2.2
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template
add area=backbone interfaces=ether2,ether3 type=ptp comment="link inter-router"
add area=backbone interfaces=ether1 passive comment="LAN"
```

**R3** — router ID `3.3.3.3`:
```routeros
/routing ospf instance add name=ospf1 version=2 router-id=3.3.3.3
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template
add area=backbone interfaces=ether2,ether3 type=ptp comment="link inter-router"
add area=backbone interfaces=ether1 passive comment="LAN"
```

`ether1 passive` berarti subnet LAN tetap diiklankan ke OSPF, tetapi router tidak mengirim Hello OSPF ke host biasa. `type=ptp` dipakai untuk link antar-router point-to-point.

### 3. Verifikasi OSPF

Pada tiap router:
```routeros
/routing ospf neighbor print
/routing ospf interface print
/ip route print where ospf
```

Yang dicari:
- Neighbor router tetangga muncul dan state mencapai `Full`.
- Interface inter-router ikut OSPF.
- Route OSPF muncul pada routing table.
- LAN tiap router dikenal oleh router lain.

Jika neighbor belum Full, cek IP kedua ujung link, status interface, area ID, router ID, dan template interface.

### 4. Uji convergence saat link R1–R3 putus

1. Dari Laterano jalankan:
   ```bash
   ping -O 10.126.20.10
   ```
2. Suspend link R1–R3 di GNS3.
3. Amati apakah traffic pulih melalui R2, berapa paket hilang, dan berapa lama.
4. Saat link suspend, periksa:
   ```routeros
   # R1
   /ip route print
   /routing ospf neighbor print
   ```
5. Resume link dan amati direct path kembali.

**Apa yang dibuktikan?** OSPF mendeteksi perubahan topologi, menyebarkan informasi baru, menghitung ulang jalur, lalu memasang route baru. Waktu sampai jaringan stabil disebut convergence time.

**Bukti C.2**
- Neighbor OSPF R1/R2/R3.
- Interface OSPF.
- Routing table route OSPF.
- Ping berkelanjutan ketika link R1–R3 suspend.
- Route dan neighbor saat link gagal.
- Jalur kembali setelah link di-resume.

---

# BAGIAN D — DUA BACKBONE DAN SKALABILITAS

## D.1 Migrasi ke backbone ganda

### Tujuan

Mengganti link point-to-point antar-router dengan dua shared Ethernet segment: Backbone A dan Backbone B. Semua router terhubung ke kedua switch backbone. OSPF berjalan pada kedua segmen.

### 1. Buat koneksi fisik

- Sambungkan R1/R2/R3 `ether4` ke switch Backbone A.
- Sambungkan R1/R2/R3 `ether5` ke switch Backbone B.
- Pastikan seluruh router terhubung ke switch yang benar.

### 2. Nonaktifkan link point-to-point dan beri IP backbone

Lakukan pada R1, R2, R3.

**R1**
```routeros
/interface disable ether2,ether3
/ip address
add address=10.126.20.145/29 interface=ether4 comment="Backbone A"
add address=10.126.20.153/29 interface=ether5 comment="Backbone B"
/routing ospf interface-template
add area=backbone interfaces=ether4,ether5 comment="backbone ganda"
```

**R2** — alamat `.146` dan `.154`:
```routeros
/interface disable ether2,ether3
/ip address
add address=10.126.20.146/29 interface=ether4 comment="Backbone A"
add address=10.126.20.154/29 interface=ether5 comment="Backbone B"
/routing ospf interface-template
add area=backbone interfaces=ether4,ether5 comment="backbone ganda"
```

**R3** — alamat `.147` dan `.155`:
```routeros
/interface disable ether2,ether3
/ip address
add address=10.126.20.147/29 interface=ether4 comment="Backbone A"
add address=10.126.20.155/29 interface=ether5 comment="Backbone B"
/routing ospf interface-template
add area=backbone interfaces=ether4,ether5 comment="backbone ganda"
```

Jangan membuat instance/area baru jika konfigurasi OSPF Bagian C masih ada. Di sini hanya menambahkan interface-template backbone.

### 3. Periksa neighbor, DR, dan BDR

Pada R1/R2/R3:
```routeros
/routing ospf neighbor print
/routing ospf interface print
```

Catat untuk Backbone A dan Backbone B:
- router mana yang berstatus DR;
- router mana yang berstatus BDR;
- neighbor mana yang Full.

DR/BDR dipilih per segmen broadcast. Jangan mengasumsikan router yang sama pasti menjadi DR pada kedua backbone; catat hasil aktual.

### 4. Periksa ECMP

Sebelum melihat route IF-106, prediksi apakah R1 memiliki satu atau lebih next hop dengan cost sama. Kemudian:
```routeros
/ip route print detail where dst-address=10.126.20.0/26
```

Jika OSPF menemukan dua jalur dengan cost sama, route dapat menampilkan beberapa gateway (ECMP). Jika cost tidak sama, jalur dengan cost terbaik yang dipilih.

### 5. Uji kegagalan satu backbone

1. Dari Laterano:
   ```bash
   ping -O 10.126.20.10
   ```
2. Suspend link R3 ke Backbone A di GNS3.
3. Amati perubahan gateway/route di R1 dan jumlah paket yang hilang.
4. Resume link tersebut.

### 6. Ubah DR priority R1 menjadi 0

Di R1:
```routeros
/routing ospf interface-template set [find where comment="backbone ganda"] priority=0
/interface disable ether4,ether5
/interface enable ether4,ether5
/routing ospf interface print
```

Disable/enable dilakukan agar pemilihan DR/BDR terjadi ulang. Priority 0 membuat R1 tidak ikut terpilih sebagai DR/BDR. Catat perubahan peran yang benar-benar muncul.

**Bukti D.1**
- Neighbor OSPF dan DR/BDR pada kedua backbone.
- Routing table detail menuju IF-106, termasuk gateway/ECMP.
- Ping saat salah satu link backbone suspend dan route yang berubah.
- Status DR/BDR setelah priority R1 menjadi 0.

## D.2 Tambahkan Ruang Baca dengan R4

### Tujuan

Menambahkan LAN baru tanpa mengubah konfigurasi R1, R2, dan R3. R4 cukup bergabung ke kedua backbone dan mengiklankan LAN Ruang Baca melalui OSPF.

### 1. Pikirkan perbandingan static route

Dengan static routing dan dua jalur backbone, setiap router lama harus diberi route ke LAN Ruang Baca (serta route backup sesuai desain), dan R4 harus diberi route menuju LAN lama. Dynamic routing mengurangi pekerjaan manual karena R4 mengiklankan subnet LAN-nya ke OSPF.

### 2. Tambahkan node dan kabel

- R4 MikroTik CHR dengan 5 adapter.
- `ether1` R4 → switch Ruang Baca.
- `ether4` R4 → Backbone A.
- `ether5` R4 → Backbone B.
- Higashi → switch Ruang Baca.

### 3. Konfigurasi R4

```routeros
/ip dhcp-client remove [find]
/system identity set name=R4
/ip address
add address=10.126.20.193/26 interface=ether1 comment="LAN Ruang Baca"
add address=10.126.20.148/29 interface=ether4 comment="Backbone A"
add address=10.126.20.156/29 interface=ether5 comment="Backbone B"
/routing ospf instance add name=ospf1 version=2 router-id=4.4.4.4
/routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
/routing ospf interface-template
add area=backbone interfaces=ether4,ether5 comment="backbone ganda"
add area=backbone interfaces=ether1 passive comment="LAN"
```

### 4. Konfigurasi Higashi

```bash
ip addr add 10.126.20.200/26 dev eth0
ip route add default via 10.126.20.193
```

### 5. Buktikan R1 belajar subnet baru tanpa diubah

Di R1:
```routeros
/ip route print where dst-address=10.126.20.192/26
```

Harus muncul route OSPF menuju Ruang Baca. Lalu dari Laterano:
```bash
mtr -r -c 5 10.126.20.200
```

Jangan menambahkan static route pada R1/R2/R3. Bukti utamanya adalah R1 otomatis mengenal `10.126.20.192/26` setelah R4 menjalankan OSPF.

### 6. Simpan konfigurasi akhir router

Pada masing-masing R1, R2, R3, R4:
```routeros
/export file=case4-R1
```
Ganti nama file sesuai router: `case4-R2`, `case4-R3`, `case4-R4`. File export `.rsc` menjadi bukti konfigurasi.

**Bukti D.2**
- Topologi R4 terhubung ke LAN Ruang Baca dan dua backbone.
- `/ip address print` R4.
- Neighbor OSPF R4.
- Higashi memiliki IP dan default gateway yang benar.
- R1 route OSPF ke `10.126.20.192/26`.
- MTR Laterano → Higashi.
- File export dari keempat router.

---

# TROUBLESHOOTING — KALAU TIDAK JALAN

## 1. Ping ke gateway LAN sendiri gagal
Cek:
```bash
ip addr
ip route
```
Lalu cek kabel host–switch–router, IP/mask host, dan IP `ether1` router. Gateway host harus berada di subnet yang sama.

## 2. Ping lintas subnet gagal
Cek berurutan:
1. IP host dan default gateway.
2. `/ip address print` pada router.
3. `/ip route print` pada setiap router di jalur pergi.
4. Route balik di router tujuan.
5. Next hop benar-benar satu subnet dengan interface keluar.
6. Interface/link aktif.

## 3. Static route tidak aktif
Pastikan alamat gateway adalah IP router tetangga yang reachable melalui link langsung. Periksa prefix destination dan status interface.

## 4. OSPF neighbor tidak muncul / tidak Full
Periksa:
- IP kedua ujung link dan prefix cocok.
- Interface aktif dan kabel benar.
- Area ID sama pada link yang sama.
- Router ID berbeda.
- Interface dimasukkan ke interface-template.
- Kedua router menjalankan versi OSPF yang kompatibel.
- Jangan jadikan link antar-router passive.

## 5. OSPF route tidak muncul
Periksa neighbor Full, interface-template, dan apakah subnet LAN diiklankan. LAN host biasanya dimasukkan sebagai passive interface, bukan dibiarkan tidak diiklankan.

## 6. MTR/ping hanya menunjukkan sebagian hop
Itu bisa berarti jalur pergi/pulang tidak simetris, router tidak membalas probe, atau route di salah satu arah belum benar. Periksa routing table router, jangan hanya melihat host asal.

## 7. Perintah “already have such address” atau konfigurasi dobel
Cek dulu:
```routeros
/ip address print
/ip route print
/routing ospf instance print
/routing ospf area print
/routing ospf interface-template print
```
Jangan mengulang `add` secara membabi buta. Hapus atau ubah entri yang memang salah berdasarkan comment/nomor entri.

---

# CHECKLIST PRAKTIKUM DAN SCREENSHOT

| Bagian | Bukti yang harus disiapkan |
|---|---|
| A.1 | Tabel VLSM lengkap: prefix, network, broadcast, host range |
| A.2 | Tabel IP interface dan routing table R1/R2/R3 |
| A.3 | Perhitungan route per router, total, backup, dan dampak tambah router |
| B.1 | IP interface, connected route, ping link langsung dan ping lintas subnet sebelum static route |
| B.2 | Prediksi return route, ping/traceroute, route table, 3 hasil MTR, capture TTL/protocol/checksum |
| B.3 | Summary route, hasil LPM `/32`, bukti routing loop dan TTL, konektivitas setelah restore |
| C.1 | Direct route, backup distance, MTR, asymmetric path, ping/link failure, check-gateway |
| C.2 | OSPF neighbor/interface/route, ping saat link gagal, convergence dan route setelah failover |
| D.1 | DR/BDR dua backbone, ECMP/gateway, uji suspend satu backbone, priority 0 dan re-election |
| D.2 | Konfigurasi R4, Higashi, route OSPF baru di R1, MTR, export empat router |

## Kebiasaan penting saat ujian/praktikum

1. **Baca dulu pertanyaan: yang diminta konfigurasi, prediksi, pengamatan, atau pembuktian?**
2. Kalau instruksi meminta prediksi, tulis prediksi sebelum menjalankan command.
3. Setelah konfigurasi, selalu verifikasi dengan `print`, `ping`, `mtr`, atau capture yang sesuai.
4. Screenshot harus memperlihatkan command dan hasilnya, bukan hanya topologi.
5. Beri comment pada static route agar mudah dicari dan diubah:
   ```routeros
   comment="IF-106 via R3"
   ```
6. Setelah eksperimen B.3/C.1, pulihkan route/interface ke kondisi normal sebelum lanjut.
7. Catat hasil aktual (gateway, TTL, neighbor, DR/BDR, paket hilang, durasi). Jangan mengarang angka downtime karena bergantung pada simulasi dan waktu deteksi.

## Ringkasan hafalan satu menit

- **VLSM:** subnet besar dulu, hitung host, blok harus sejajar.
- **Connected route:** otomatis muncul karena IP dipasang pada interface.
- **Static route:** admin menentukan destination dan next hop.
- **Return route:** reply juga harus tahu jalan pulang.
- **Summarization:** beberapa subnet berurutan, next hop sama → satu route.
- **LPM:** prefix paling panjang menang.
- **Loop:** route salah membuat paket berputar; TTL habis.
- **OSPF:** instance → area → interface-template; cek neighbor Full dan route OSPF.
- **Passive LAN:** subnet tetap diiklankan, Hello tidak dikirim ke host.
- **DR/BDR:** pemimpin/cadangan pada segmen broadcast OSPF.
- **ECMP:** beberapa next hop dengan cost sama.
- **Convergence:** waktu sampai route kembali stabil setelah perubahan.
- **R4:** cukup iklankan LAN baru melalui OSPF; router lama belajar otomatis.

---

## Catatan batas sumber

Pra-Praktikum yang diberikan secara eksplisit merinci konfigurasi Bagian A–D untuk skenario R1–R4. README materi menjelaskan teori subnetting, static routing, RIP, OSPF, redundancy, dan convergence. Jika pengawas memberikan variasi alamat atau topologi, gunakan prinsip dan tabel alokasi pada soal yang sedang diberikan; jangan menyalin alamat contoh ini secara membabi buta.


# TUGAS MANDIRI — CONTOH PENYELESAIAN DI KERTAS

Bagian ini mengikuti soal terakhir pada Pra-Praktikum. Tidak perlu GNS3. Contoh memakai prefix `10.126`; jika prefix yang diberikan berbeda, ganti bagian prefix saja.

## 1. VLSM untuk lima segmen

Alokasi induk: `10.126.50.0/24`.

| Segmen | Kebutuhan | Prefix | Network | Host usable | Broadcast |
|---|---:|---:|---|---|---|
| Lab Jaringan | ±60 | `/26` | `10.126.50.0` | `.1 – .62` | `.63` |
| Lab Sistem Cerdas | ±25 | `/27` | `10.126.50.64` | `.65 – .94` | `.95` |
| Ruang Server | ±6 | `/29` | `10.126.50.96` | `.97 – .102` | `.103` |
| Link RA–RB | 2 | `/30` | `10.126.50.104` | `.105 – .106` | `.107` |
| Link RB–RC | 2 | `/30` | `10.126.50.108` | `.109 – .110` | `.111` |

Urutan alokasi berdasarkan kebutuhan host: 60, 25, 6, 2, 2. Sisa alamat `/24` belum digunakan.

## 2. Asumsi pemetaan router

Soal sinopsis tidak menuliskan secara eksplisit router mana melayani ruang mana. Untuk contoh jawaban ini diasumsikan:
- RA melayani Lab Jaringan.
- RB melayani Lab Sistem Cerdas.
- RC melayani Ruang Server.

Jika gambar/topologi pengawas menentukan pemetaan berbeda, tukar pemetaan interface LAN sesuai gambar, tetapi pertahankan subnet dan prinsip link.

## 3. Rencana alamat interface

| Router | ether1 (LAN) | ether2 | ether3 |
|---|---|---|---|
| RA | `10.126.50.1/26` | `10.126.50.105/30` ke RB | — |
| RB | `10.126.50.65/27` | `10.126.50.106/30` ke RA | `10.126.50.109/30` ke RC |
| RC | `10.126.50.97/29` | `10.126.50.110/30` ke RB | — |

Router bernomor/berhuruf lebih awal mendapat alamat host pertama pada link RA–RB; RB mendapat host kedua. Pada link RB–RC, RB mendapat host pertama dan RC host kedua.

## 4. Rencana routing table

**RA**
- Connected: Lab Jaringan `10.126.50.0/26`, Link RA–RB `10.126.50.104/30`.
- Static: Lab Sistem Cerdas `10.126.50.64/27` via `10.126.50.106`.
- Static: Ruang Server `10.126.50.96/29` via `10.126.50.106`.

**RB**
- Connected: Lab Sistem Cerdas `10.126.50.64/27`, Link RA–RB `10.126.50.104/30`, Link RB–RC `10.126.50.108/30`.
- Static: Lab Jaringan `10.126.50.0/26` via `10.126.50.105`.
- Static: Ruang Server `10.126.50.96/29` via `10.126.50.110`.

**RC**
- Connected: Ruang Server `10.126.50.96/29`, Link RB–RC `10.126.50.108/30`.
- Static: Lab Jaringan `10.126.50.0/26` via `10.126.50.109`.
- Static: Lab Sistem Cerdas `10.126.50.64/27` via `10.126.50.109`.

Semua static route di atas harus memiliki pasangan arah balik di router yang sesuai. Misalnya, agar host Lab Jaringan dapat berkomunikasi dengan Ruang Server, RA harus tahu tujuan Ruang Server melalui RB; RB harus tahu Ruang Server melalui RC; RC harus tahu Lab Jaringan melalui RB; dan RB harus tahu Lab Jaringan melalui RA.

## 5. Route summarization yang mungkin

Pada RA, dua tujuan di belakang RB:
- Lab Sistem Cerdas `10.126.50.64/27`
- Ruang Server `10.126.50.96/29`

keduanya berada dalam rentang `10.126.50.64–10.126.50.127`, sehingga dapat dicakup oleh summary `10.126.50.64/26` via RB. Summary ini juga mencakup alamat yang belum dipakai dalam blok tersebut.

Jangan menggabungkan tujuan yang next hop-nya berbeda. Pada RB, Lab Jaringan menuju RA sedangkan Ruang Server menuju RC, sehingga keduanya tidak dapat dijadikan satu summary dengan satu next hop.

## 6. Jika link RC–RA ditambahkan sebagai backup

Jika link langsung RC–RA ditambahkan, router di ujung link memperoleh jalur alternatif. Jumlah static route tambahan bergantung pada desain backup yang diminta:
- RA perlu route alternatif menuju Ruang Server melalui RC (jika jalur utama sebelumnya melalui RB).
- RC perlu route alternatif menuju Lab Jaringan melalui RA (jika jalur utama sebelumnya melalui RB).
- Router yang tidak menjadi ujung link tidak otomatis memerlukan route baru hanya karena kabel RC–RA dipasang, kecuali desain meminta mereka menggunakan/mengetahui jalur alternatif tersebut.

Inilah alasan dynamic routing lebih mudah dikelola saat jumlah router dan jalur alternatif bertambah: router bertukar informasi dan menghitung ulang jalur tanpa administrator menambah route satu per satu pada seluruh perangkat.
