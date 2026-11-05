# Modul-Jarkom3

# Pra-Praktikum Modul 3 — Reliable Data Transfer TCP & Static Routing

**Pre-requisite:**

1. GNS3 VM dan GNS3-GUI telah terpasang dan dapat dijalankan.
2. Appliance `netics-pc`/`netics-server` dan `netics-pc-desktop` tersedia di GNS3.

**Cakupan Modul:** Three-way handshake, sequence & acknowledgment number, retransmission dan duplicate ACK, TCP congestion control (CUBIC vs Reno), perbandingan TCP vs UDP, static routing.

> [!NOTE]
> - IP Prefix disesuaikan dengan pembagian yang sudah diumumkan sebelumnya (pada pengerjaan ini: `10.182`).
> - Tolong dipahami materi modul 3 dan sebelum-sebelumnya, karena copas saja tidak cukup.

## Pemetaan Node

| Node | Appliance | Peran | IP Address |
| --- | --- | --- | --- |
| RhodesIsland | netics-pc/netics-server | Router ke NAT untuk mendapatkan akses internet, dan penghubung antara client dan web server, sekaligus titik penerapan gangguan jaringan (`tc netem`) | eth0: DHCP<br>eth1: `10.182.1.1/24`<br>eth2: `10.182.2.1/24` |
| Lungmen | netics-pc/netics-server | Router | eth0: `10.182.1.2/24`<br>Gateway: `10.182.1.1`<br>eth1: `10.182.3.1/24` |
| Ursus | netics-pc/netics-server | Router | eth0: `10.182.2.2/24`<br>Gateway: `10.182.2.1`<br>eth1: `10.182.4.1/24` |
| Laterano | netics-pc/netics-server | Web server, menyediakan berkas uji dengan variasi ukuran (1000 B, 10.000 B, 5 MB).<br>Capture point Wireshark (sisi sender) | DHCP Client, pastikan Laterano selalu mendapat IP statis dari Yan: `10.182.4.15`<br>Subnet: `4.0/24` |
| Yan | netics-pc/netics-server | DHCP Server untuk Subnet `10.182.4.0/24` | eth0: `10.182.4.2/24`<br>Gateway: `10.182.4.1`<br>DHCP pool: `10.182.4.50` - `10.182.4.60`<br>Pakai `kea-dhcp4` |
| Columbia | netics-pc-desktop | Client uji (web client), menjalankan terminal (xterm) (`curl`/`iperf3`), dan Firefox.<br>Capture point Wireshark (sisi receiver) | DHCP Client dari Victoria |
| Victoria | netics-pc/netics-server | DHCP Server untuk Subnet `10.182.3.0/24` | eth0: `10.182.3.2/24`<br>Gateway: `10.182.3.1`<br>DHCP pool: `10.182.3.100` - `10.182.3.120` |

---

# Bagian A - Reliable Data Transfer pada Kondisi Normal

## a. Tujuan Pembelajaran

1. Memahami prinsip kerja reliable data transfer yang diimplementasikan pada protokol transport TCP.
2. Mengamati mekanisme pembentukan koneksi TCP (three-way handshake: SYN, SYN-ACK, ACK) melalui capture Wireshark.
3. Mengamati bagaimana sequence number dan acknowledgment number bekerja selama pengiriman data berlangsung.

## b. Tools yang Dibutuhkan

- GNS3 appliance `netics-pc`/`netics-server` dan `netics-pc-desktop`
- `curl` dan python HTTP server sederhana (di Laterano)
- NAT node
- DHCP menggunakan `kea-dhcp4` (modul 1)
- `iperf3`
- Wireshark
- Pastikan sudah mempelajari dan implementasi contoh static routing di modul 3

## c. Persiapan Topologi

### Langkah Kerja

1. Buat topologi sesuai yang ada pada gambar dan pemetaan node di atas.
2. Wireshark digunakan pada soal untuk capture packet di topologi GNS3.
3. Konfigurasikan setiap router yang ada dalam tabel pemetaan node agar tiap node dalam topologi dapat saling melakukan ping.

```sh
# Konfigurasi untuk RhodesIsland
auto eth0
iface eth0 inet dhcp
auto eth1
iface eth1 inet static
address 10.182.1.1
netmask 255.255.255.0
auto eth2
iface eth2 inet static
address 10.182.2.1
netmask 255.255.255.0

# Konfigurasi untuk Lungmen
auto eth0
iface eth0 inet static
address 10.182.1.2
netmask 255.255.255.0
gateway 10.182.1.1
auto eth1
iface eth1 inet static
address 10.182.3.1
netmask 255.255.255.0

# Konfigurasi untuk Ursus
auto eth0
iface eth0 inet static
address 10.182.2.2
netmask 255.255.255.0
gateway 10.182.2.1
auto eth1
iface eth1 inet static
address 10.182.4.1
netmask 255.255.255.0
```

4. Kemudian konfigurasikan static routing untuk komunikasi antara subnet `1.0/24`, `2.0/24`, `3.0/24` dan `4.0/24` agar saling terhubung.
5. Command `ip route add` dapat dikonfigurasikan otomatis melalui `init.sh`:

```sh
# RhodesIsland
$ ip route add 10.182.3.0/24 via 10.182.1.2 dev eth1
$ ip route add 10.182.4.0/24 via 10.182.2.2 dev eth2
$ iptables -t nat -A POSTROUTING -j MASQUERADE -s 10.182.0.0/16 -o eth0

# Lungmen
$ ip route add 10.182.2.0/24 via 10.182.1.1 dev eth0
$ ip route add 10.182.4.0/24 via 10.182.1.1 dev eth0

# Ursus
$ ip route add 10.182.1.0/24 via 10.182.2.1 dev eth0
$ ip route add 10.182.3.0/24 via 10.182.2.1 dev eth0

# Verifikasi pada setiap router
ip route show
```

6. Kemudian coba ping dari Victoria ke node Yan, pastikan kedua node dapat saling melakukan ping!
7. Untuk semua node, **pastikan** dapat melakukan ping ke `google.com` dan menambahkan nameserver `8.8.8.8` ke `/etc/resolv.conf`:

```sh
$ echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

8. Konfigurasikan DHCP server pada node Victoria dan Yan sesuai dengan petunjuk pemetaan node, pastikan menggunakan `kea-dhcp4` (ada di modul 1) dan tidak menggunakan `udhcpd`.

Kalian dapat hapus isi file `/etc/kea/kea-dhcp4.conf` dan mengganti isinya dengan hint berikut.

**Hint untuk Victoria**

```json
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": ["eth0"] },
    "lease-database": { "type": "memfile", "persist": true, "name": "/var/lib/kea/kea-leases4.csv" },
    "valid-lifetime": 3600,
    "subnet4": [
      {
        "id": 1,
        "subnet": "10.182.3.0/24",
        "pools": [ { "pool": "10.182.3.100 - 10.182.3.120" } ],
        "option-data": [
          { "name": "routers", "data": "10.182.3.1" },
          { "name": "domain-name-servers", "data": "8.8.8.8" }
        ]
      }
    ]
  }
}
```

**Hint untuk Yan**

```json
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": ["eth0"] },
    "lease-database": { "type": "memfile", "persist": true, "name": "/var/lib/kea/kea-leases4.csv" },
    "valid-lifetime": 3600,
    "subnet4": [
      {
        "id": 1,
        "subnet": "10.182.4.0/24",
        "pools": [ { "pool": "10.182.4.50 - 10.182.4.60" } ],
        "option-data": [
          { "name": "routers", "data": "10.182.4.1" },
          { "name": "domain-name-servers", "data": "8.8.8.8" }
        ],
        "reservations": [
          { "hw-address": "<MAC eth0 Laterano>", "ip-address": "10.182.4.15" }
        ]
      }
    ]
  }
}
```

### Langkah Kerja, Web Server (Laterano)

1. Konfigurasikan IP address Laterano secara dinamis yang diperoleh dari DHCP server node Yan, pastikan Laterano selalu mendapat IP statis dari DHCP server dengan IP `10.182.4.15`.
2. Konfigurasikan DHCP di node Victoria seperti pada tabel pemetaan dan hint.
3. Pastikan node Yan dan Laterano dapat terhubung ke internet dan di Laterano install `python3` seperti yang ada di modul 2.
4. Buat berkas `/root/myscripts/start_http.sh` yang menyiapkan direktori `myweb` dan menjalankan HTTP server sederhana bawaan Python pada port 9999:

```sh
# /root/myscripts/start_http.sh
#!/bin/sh

mkdir -p myweb
cd myweb

python3 -m http.server 9999 -d /root/myscripts/myweb
```

5. Buat berkas uji dengan variasi ukuran di direktori `/root/myscripts/myweb`:

```sh
# Berkas uji di /root/myscripts/myweb
$ dd if=/dev/random of=tes1000.bin  bs=1    count=1000
$ dd if=/dev/random of=tes10000.bin bs=10   count=1000
$ dd if=/dev/random of=tes50000.bin bs=50   count=1000
$ dd if=/dev/random of=tes5M.bin    bs=5000 count=1000
```

### Langkah Kerja, Verifikasi Awal dari Columbia

1. Konfigurasikan DHCP di node Victoria seperti pada tabel pemetaan dan hint.
2. Buka console via VNC, jalankan terminal (xterm), lakukan ping ke `10.182.4.15` untuk memastikan konektivitas dasar melalui konfigurasi static routing sudah berjalan.
3. Buka Firefox, arahkan ke `http://10.182.4.15:9999`. Setelah berkas uji dibuat di sisi server (Laterano), refresh halaman, seluruh berkas (`tes1000.bin`, `tes10000.bin`, `tes50000.bin`, `tes5M.bin`) akan terlihat pada listing.

### Indikator Keberhasilan Persiapan

| No | Parameter / bukti | Field / perintah | Yang harus dijelaskan |
| --- | --- | --- | --- |
| 1 | Topologi | GNS3 | Node dan link sesuai Gambar 0.1 serta tabel pemetaan. |
| 2 | IP & DHCP | `ip addr`; lease client | Columbia memperoleh pool `.3.x` dan Laterano memperoleh `.4.15` dari Yan. |
| 3 | Routing & NAT | `ip route`; ping | Subnet internal saling terhubung; akses internet terverifikasi jika NAT tersedia. |
| 4 | HTTP server | Browser/curl port 9999 | Berkas uji dapat diakses dari Columbia. |

---

## A.3 - Persiapan Topologi

> Langkah kerja, topologi, dan persiapan konfigurasi network (Pre-Praktikum). Seluruh pengujian pada Bagian A - D menggunakan topologi yang sama.

#### Soal A.3.1 - Topologi GNS3

> Buat topologi sesuai gambar dan tabel pemetaan node.

**Jawaban:**

Topologi dibangun sesuai gambar dan tabel pemetaan node (screenshot topologi dihapus dari versi teks ini).

#### Soal A.3.2 - Konfigurasi IP / Network & Static Routing

> Tunjukkan konfigurasi `/etc/network/interfaces` tiap router (RhodesIsland, Lungmen, Ursus), command `ip route add` (dapat dikonfigurasikan otomatis melalui `init.sh`), hasil `ip route show` pada ketiga router, serta bukti ping antara Victoria dan Yan.

**Jawaban:**

#### A. Isi Konfigurasi `/etc/network/interfaces`

- RhodesIsland

```sh
auto eth0
iface eth0 inet dhcp

auto eth1
iface eth1 inet static
address 10.182.1.1
netmask 255.255.255.0

auto eth2
iface eth2 inet static
address 10.182.2.1
netmask 255.255.255.0
```

- Lungmen

```sh
auto eth0
iface eth0 inet static
address 10.182.1.2
netmask 255.255.255.0
gateway 10.182.1.1

auto eth1
iface eth1 inet static
address 10.182.3.1
netmask 255.255.255.0
```

- Ursus

```sh
auto eth0
iface eth0 inet static
address 10.182.2.2
netmask 255.255.255.0
gateway 10.182.2.1

auto eth1
iface eth1 inet static
address 10.182.4.1
netmask 255.255.255.0
```

#### B. Hasil `ip route add` dan `ip route show` di masing-masing router

- RhodesIsland

```sh
ip route add 10.182.3.0/24 via 10.182.1.2 dev eth1
ip route add 10.182.4.0/24 via 10.182.2.2 dev eth2
ip route show
```

- Lungmen

```sh
ip route add 10.182.2.0/24 via 10.182.1.1 dev eth0
ip route add 10.182.4.0/24 via 10.182.1.1 dev eth0
ip route show
```

- Ursus

```sh
ip route add 10.182.1.0/24 via 10.182.2.1 dev eth0
ip route add 10.182.3.0/24 via 10.182.2.1 dev eth0
ip route show
```

#### Soal A.3.3 - NAT dan DNS

> Tunjukkan aturan `iptables` MASQUERADE pada RhodesIsland (`-s 10.182.0.0/16 -o eth0`), `nameserver 8.8.8.8` pada `/etc/resolv.conf` di semua node, dan bukti semua node (RhodesIsland, Lungmen, Ursus, Victoria, Yan) dapat `ping google.com`.

**Jawaban:**

#### A. Isi aturan `iptables` MASQUERADE di RhodesIsland

```sh
iptables -t nat -A POSTROUTING -j MASQUERADE -s 10.182.0.0/16 -o eth0

# Cek dengan:
iptables -t nat -S POSTROUTING
```

#### B. Isi `etc/resolve.conf` dan bukti `ping google.com` di semua node

Dijalankan di RhodesIsland, Lungmen, Ursus, Victoria, dan Yan:

```sh
echo "nameserver 8.8.8.8" > /etc/resolv.conf
cat /etc/resolv.conf
ping -c 4 google.com
```

#### Soal A.3.4 - DHCP Server (kea-dhcp4) pada Yan dan Victoria

> Tunjukkan `/etc/kea/kea-dhcp4.conf` pada Yan (termasuk reservasi Laterano `10.182.4.15`) dan Victoria menggunakan kea-dhcp4 (bukan udhcpd), bukti `kea-dhcp4` berjalan, serta bukti lease client (`udhcpc -i eth0` dan `ip -br addr`) pada Laterano dan Columbia.

**Jawaban:**

#### A. Isi `/etc/kea/kea-dhcp4.conf` pada Yan dan Victoria

##### A.1 Instalasi kea-dhcp4

Dilakukan di Yan dan Victoria:

```sh
apk update
apk add kea-dhcp4
```

##### A.2 Pengisian `/etc/kea/kea-dhcp4.conf`

- Victoria

```json
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": ["eth0"] },
    "lease-database": { "type": "memfile", "persist": true, "name": "/var/lib/kea/kea-leases4.csv" },
    "valid-lifetime": 3600,
    "subnet4": [
      {
        "id": 1,
        "subnet": "10.182.3.0/24",
        "pools": [ { "pool": "10.182.3.100 - 10.182.3.120" } ],
        "option-data": [
          { "name": "routers", "data": "10.182.3.1" },
          { "name": "domain-name-servers", "data": "8.8.8.8" }
        ]
      }
    ]
  }
}
```

- Yan

a) Memeriksa MAC Address dari Laterano:

```sh
ip link show eth0
```

b) Isi config

```json
{
  "Dhcp4": {
    "interfaces-config": { "interfaces": ["eth0"] },
    "lease-database": { "type": "memfile", "persist": true, "name": "/var/lib/kea/kea-leases4.csv" },
    "valid-lifetime": 3600,
    "subnet4": [
      {
        "id": 1,
        "subnet": "10.182.4.0/24",
        "pools": [ { "pool": "10.182.4.50 - 10.182.4.60" } ],
        "option-data": [
          { "name": "routers", "data": "10.182.4.1" },
          { "name": "domain-name-servers", "data": "8.8.8.8" }
        ],
        "reservations": [
          { "hw-address": "<MAC eth0 Laterano>", "ip-address": "10.182.4.15" }
        ]
      }
    ]
  }
}
```

#### B. Bukti `kea-dhcp4` berjalan

Dijalankan di Victoria dan Yan:

```sh
kea-dhcp4 -d -c /etc/kea/kea-dhcp4.conf
```

#### C. Bukti lease client (`udhcpc -i eth0` dan `ip -br addr`) pada Laterano

```sh
udhcpc -i eth0
ip -br addr
```

Penjelasan: IP yang dihasilkan dari DHCP harus `10.182.4.15`, maka sudah benar untuk MAC Address Laterano yang dituliskan di file `kea-dhcp4.conf` milik Yan.

#### Soal A.3.5 - HTTP Server pada Laterano

> Pastikan Yan dan Laterano terhubung ke internet dan python3 sudah terpasang di Laterano (seperti modul 2). Tunjukkan `/root/myscripts/start_http.sh` yang dijalankan di tmux (split window) serta pembuatan berkas uji dengan `dd` (`tes1000.bin`, `tes10000.bin`, `tes50000.bin`, `tes5M.bin`).

**Jawaban:**

#### A. Cek konektivitas Yan dan Laterano melalui `ping google.com` dan menginstall python3 di Laterano

- ping google.com

```sh
ping -c 4 google.com
```

- Install python3

```sh
apk add python3
```

#### B. Membuat file `/root/myscripts/start_http.sh` di Laterano

```sh
#!/bin/sh

mkdir -p myweb
cd myweb

python3 -m http.server 9999 -d /root/myscripts/myweb
```

#### C. Jalankan `start_http.sh` di tmux dan membuat berkas uji dengan `dd` (`tes1000.bin`, `tes10000.bin`, `tes50000.bin`, `tes5M.bin`)

```sh
# Pane 1 (tmux): jalankan HTTP server
chmod +x /root/myscripts/start_http.sh
./start_http.sh

# Pane 2 (tmux): buat berkas uji
cd /root/myscripts/myweb
dd if=/dev/random of=tes1000.bin  bs=1    count=1000
dd if=/dev/random of=tes10000.bin bs=10   count=1000
dd if=/dev/random of=tes50000.bin bs=50   count=1000
dd if=/dev/random of=tes5M.bin    bs=5000 count=1000
```

#### Soal A.3.6 - Verifikasi Awal dari Columbia

> Dari Columbia (VNC/xterm), tunjukkan lease DHCP dari Victoria, `ping` ke `10.182.4.15`, dan Firefox ke `http://10.182.4.15:9999` yang menampilkan seluruh berkas uji pada listing.

**Jawaban:**

#### A. Lease DHCP dari Victoria di XTerm Columbia

```sh
udhcpc -i eth0
ip -br addr
```

#### B. Ping ke `10.182.4.15` (Laterano)

```sh
ping -c 4 10.182.4.15
```

#### C. Listing berkas uji

- Melalui curl

```sh
curl http://10.182.4.15:9999
```

- Ketik url di Firefox: `http://10.182.4.15:9999` (listing menampilkan `tes1000.bin`, `tes10000.bin`, `tes50000.bin`, `tes5M.bin`).

---

## d. Problem A.4 - Observasi Transfer Tanpa Gangguan (Skenario 1)

Dengan topologi yang sudah siap, amati mekanisme reliable data transfer TCP pada kondisi jaringan normal (tanpa gangguan).

**Soal:**

Pastikan Columbia berhasil membangun koneksi TCP dengan Laterano saat mengunduh berkas `tes10000.bin`. Amati urutan awal pembentukan koneksi melalui Wireshark dan identifikasi identitas endpoint yang terlibat.

### Langkah Kerja

1. Pastikan kondisi interface eth1 dan eth2 RhodesIsland belum diberi gangguan apa pun (`tc qdisc` default/tanpa aturan):

```sh
tc qdisc show dev eth1
tc qdisc show dev eth2
# Hapus qdisc gangguan jika memang ada:
tc qdisc del dev eth1 root
tc qdisc del dev eth2 root
```

2. Di antara Columbia dan Switch1, mulai capture pada interface eth0 (membuka Wireshark), terapkan filter tampilan:

```
tcp.port == 9999
```

3. Lakukan start capture menggunakan Wireshark di Laterano dengan cara yang sama seperti di Columbia.
4. Sebelum menjalankan download, tuliskan dulu prediksi: nilai sequence number awal (initial sequence number) yang akan dikirim client pada paket SYN, apakah dimulai dari 0, atau nilai lain? Catat prediksi ini sebelum melihat hasil capture.
5. Lakukan start capture menggunakan Wireshark di **Laterano** dengan cara yang sama seperti di Columbia.
6. Buka terminal (xterm) di Columbia, lakukan download berkas `tes10000.bin` sambil mengamati aktivitas pada Wireshark:

```sh
curl -o /dev/null -s -w "size=%{size_download}B speed=%{speed_download}B/s total=%{time_total}s\n" http://10.182.4.15:9999/tes10000.bin
```

7. Hentikan capture Wireshark di Columbia setelah download selesai, lalu telusuri paket-paket yang tercatat (hasil capture dari port eth Columbia dan Laterano). Bandingkan prediksi sequence number awal dengan nilai aktual pada paket SYN, jelaskan jika berbeda, mengapa initial sequence number tidak dimulai dari nol.
8. **Verifikasi manual SEQ/panjang payload:** pilih satu segmen data di tengah stream (bukan segmen pertama atau terakhir), catat nilai `tcp.seq` dan panjang payload-nya (`tcp.len`), lalu buktikan bahwa segmen berikutnya memiliki `tcp.seq` = `tcp.seq` segmen ini + `tcp.len` segmen ini.
9. Catat nilai `tcp.analysis.ack_rtt` pada salah satu ACK (di Laterano), lalu diskusikan: jika nilai RTT ini dipakai TCP untuk menetapkan retransmission timer, jelaskan mengapa timer yang terlalu pendek maupun terlalu panjang sama-sama bermasalah bagi kinerja transfer.

> [!NOTE]
> Skenario 1: tanpa gangguan `tc/netem` pada eth1 dan eth2 RhodesIsland. Download `tes10000.bin` dari Columbia ke Laterano (port 9999). Capture Wireshark di **Columbia** dan **Laterano** dengan filter `tcp.port == 9999`.

#### Soal A.4.1 - Persiapan & Prediksi

> Pastikan tidak ada qdisc gangguan (`tc qdisc show`/`tc qdisc del`). Tuliskan **prediksi** ISN client pada paket SYN sebelum melihat capture.

**Jawaban:**

#### A. Prediksi ISN Client

Prediksi saya: ISN client pada paket SYN tidak dimulai dari 0, melainkan nilai acak 32-bit. Alasannya, ISN diacak untuk mencegah tebakan sequence number oleh pihak luar dan mencegah tercampurnya segmen lama dari koneksi sebelumnya dengan koneksi baru.

#### B. Lakukan `tc qdisc show` di RhodesIsland

```sh
tc qdisc show dev eth1
tc qdisc show dev eth2
```

#### Soal A.4.2 - Tabel Observasi Three-Way Handshake

> [!NOTE]
> Untuk parameter 6-8 gunakan capture **Laterano**; untuk 9-10 gunakan capture **Columbia**.

> Isi tabel berikut berdasarkan capture Wireshark.

| No | Parameter/Aspek | Field Wireshark | Instruksi | Jawaban |
| --- | --- | --- | --- | --- |
| 1 | IP sumber dan tujuan | `ip.src`, `ip.dst` | Pastikan koneksi terjadi antara client (pengirim request) dan server (penerima request). | — |
| 2 | Proses handshake (3 paket awal) | `tcp.flags.syn`, `tcp.flags.ack` | Amati tiga paket pertama: SYN → SYN-ACK → ACK. | — |
| 3 | Prediksi vs aktual: sequence number awal client | `tcp.seq` (SYN) | Bandingkan prediksi sebelum capture dengan nilai aktual, jelaskan mengapa ISN acak, bukan nol. | — |
| 4 | Sequence number awal server | `tcp.seq` (SYN-ACK) | Tunjukkan byte awal dari sisi server. | — |
| 5 | ACK setelah handshake | `tcp.ack` | Tunjukkan bahwa client mengonfirmasi SYN-ACK dari server. | — |
| 6 | Verifikasi SEQ segmen berikutnya = SEQ + panjang payload | `tcp.seq`, `tcp.len` | Pilih satu segmen data di tengah stream (bukan segmen pertama atau terakhir), lalu buktikan secara manual bahwa segmen berikutnya memiliki `tcp.seq` = `tcp.seq` + `tcp.len` segmen ini. | `tcp.seq` segmen ini: 1445109190 (Paket 10)<br>`tcp.len` segmen ini: 1448<br>`tcp.seq` berikutnya: 1445110638 (Paket 11)<br>Perhitungan: 1445109190 + 1448 = 1445110638 |
| 7 | Pola ACK penerima | `tcp.ack` | Tunjukkan bahwa ACK bertambah kumulatif, menandakan data diterima secara berurutan. | — |
| 8 | RTT rata-rata dan implikasinya terhadap timer | `tcp.analysis.ack_rtt` | Catat RTT pada salah satu ACK (di Laterano), lalu diskusikan mengapa timer yang terlalu pendek maupun terlalu panjang sama-sama bermasalah bagi kinerja transfer. | RTT pada Paket 3 dan Paket 16 dicatat dari field `tcp.analysis.ack_rtt`.<br> Jika RTT terlalu pendek maka RTO habis sebelum ACK sempat tiba, padahal segmen tidak hilang, hanya masih dalam perjalanan.<br> Jika terlalu panjang maka segmen yang benar-benar hilang baru dikirim ulang setelah menunggu lama. |
| 9 | Status retransmisi | `tcp.analysis.retransmission` | Semestinya tidak ada retransmisi, menandakan link sehat. | Tidak terlihat retransmisi -> link sehat |
| 10 | Pola penutupan koneksi | `tcp.flags.fin`, `tcp.flags.ack` | Tunjukkan bahwa TCP menutup koneksi melalui four-way handshake (FIN/ACK). | Terlihat bahwa koneksi ditutuo melalui 4-way handshake. Setiap FIN dikonfirmasi dengan ACK bernomor FIN + 1, karena FIN memakai satu nomor urut. Penutupan dilakukan per arah (full-duplex), sehingga masing-masing sisi menutup sendiri-sendiri. |

#### A.4.2 - Pertanyaan Analisis / Refleksi

> 1. Apa fungsi utama sequence number dan acknowledgment number dalam TCP?
>
> 2. Mengapa ACK pada TCP bersifat kumulatif, bukan per-paket?
>
> 3. Apa yang akan terjadi jika salah satu segmen data hilang di tengah pengiriman?
>
> 4. Apa bukti dari hasil Wireshark bahwa TCP berhasil melakukan transfer data secara reliable?

**Jawaban:**

1. Sequence number memberi nomor urut pada setiap byte data yang dikirim, jadi penerima bisa menyusun data sesuai urutan dan mendeteksi duplikat atau byte yang hilang. Acknowledgment number memberi tahu pengirim byte mana yang sudah diterima dan byte berikutnya yang diharapkan.

2. Satu ACK dengan nomor N berarti “semua byte sebelum N sudah diterima”. Cara ini lebih efisien karena satu ACK bisa mengonfirmasi banyak segmen sekaligus, sehingga lalu lintas ACK lebih sedikit. ACK yang hilang juga tidak masalah, sebab ACK berikutnya otomatis mencakupnya.

3. Penerima tidak bisa memajukan ACK karena ada celah. Setiap segmen yang datang setelahnya memicu duplicate ACK dengan nomor yang sama. Setelah 3 dupACK, pengirim langsung mengirim ulang segmen yang hilang (fast retransmit). Kalau tidak ada dupACK, pengirim menunggu timer habis (timeout) lalu mengirim ulang. Setelah itu TCP mengurangi cwnd (laju kirim turun), jadi throughput menurun.

4. Dari capture wireshark yang sudah dilakukan:
- Handshake lengkap: SYN → SYN-ACK → ACK.
- Nomor urut kontinu tanpa celah, misalnya 1445109190 + 1448 = 1445110638.
- ACK dari client naik terus (kumulatif), tidak pernah turun.
- Total data yang diterima sesuai ukuran file: 6 × 1448 + 1312 = 10.000 byte.
- Tidak ada tcp.analysis.retransmission (hasil filter kosong), jadi tidak ada data yang hilang.
- Koneksi ditutup rapi dengan FIN/ACK, dan Ack=10206 pada FIN client mengonfirmasi seluruh data sudah diterima.

---

# Bagian B - Retransmission & Duplicate ACK

## a. Tujuan Pembelajaran

1. Mengamati bagaimana TCP mendeteksi segmen yang hilang melalui duplicate ACK.
2. Mengamati mekanisme retransmisi TCP, baik melalui fast retransmit maupun timeout.
3. Memahami dampak packet loss terhadap throughput dan waktu transfer.

## b. Tools yang Dibutuhkan

- `tc` (traffic control) dengan modul `netem`, dijalankan di RhodesIsland
- Wireshark, `curl` (sama seperti Bagian A)

## c. Problem B.3 Skenario 2: Transfer dengan Packet Loss 1%

**Soal:**

Terapkan simulasi packet loss 1% pada interface eth1 dan eth2 RhodesIsland, lalu amati bagaimana TCP merespons melalui retransmisi dan duplicate ACK.

### Langkah Kerja

1. Di RhodesIsland, hapus qdisc yang mungkin sudah ada, lalu terapkan simulasi packet loss 1% pada kedua interface eth1 dan eth2 pada RhodesIsland.

> [!NOTE]
> Error saat menjalankan `tc qdisc del` saat pertama kali adalah normal dan expected.

```sh
$ tc qdisc del dev eth1 root
$ tc qdisc del dev eth2 root
$ tc qdisc add dev eth1 root netem loss 1%
$ tc qdisc add dev eth2 root netem loss 1%
$ tc qdisc show dev eth1
$ tc qdisc show dev eth2
```

2. Di antara Columbia dan Switch1, mulai capture pada interface eth0 (membuka Wireshark), terapkan filter tampilan:

```
tcp.port == 9999
```

3. Lakukan start capture menggunakan Wireshark di Laterano dengan cara yang sama seperti di Columbia.
4. Lakukan download berkas `tes5M.bin` (ukuran lebih besar agar peluang loss lebih terlihat) sambil mengamati aktivitas Wireshark:

```sh
$ curl -o /dev/null -s -w "size=%{size_download}B speed=%{speed_download}B/s total=%{time_total}s\n" http://10.182.4.15:9999/tes5M.bin
```

5. Seperti skenario 1, hentikan capture Wireshark di Columbia setelah download selesai, lalu telusuri paket-paket yang tercatat (hasil capture dari port eth Columbia dan Laterano).
6. Verifikasi manual segmen retransmisi (di Laterano): temukan satu paket dengan tanda `tcp.analysis.retransmission`, lalu bandingkan nilai `tcp.seq` dan `tcp.len` paket tersebut dengan segmen aslinya yang hilang (biasanya segmen sebelumnya dengan SEQ yang sama), buktikan bahwa retransmisi mengirim ulang byte yang identik, bukan data baru.
7. Bandingkan prediksi awal dengan hasil capture, cocokkan dengan pola duplicate ACK yang benar-benar muncul (ACK dengan nomor yang sama berulang, bukan NAK).
8. Ukur selisih waktu (`frame.time_relative`) antara duplicate ACK ketiga dan paket retransmisi yang menyusul (di Laterano), bandingkan seberapa cepat fast retransmit terjadi dibanding jika harus menunggu retransmission timer habis (timeout).

> [!NOTE]
> Skenario 2: packet loss 1% pada eth1 dan eth2 RhodesIsland. Download `tes5M.bin`. Untuk parameter 2 dan 5 gunakan capture **Columbia**; selain itu gunakan capture **Laterano**.

#### Soal B.1 - Persiapan Gangguan & Prediksi

> Terapkan `tc qdisc add dev eth1/eth2 root netem loss 1%`. Tuliskan prediksi reaksi penerima terhadap loss sebelum melihat capture.

**Jawaban:**

Prediksi saya: Ketika sebuah segmen hilang di tengah pengiriman, penerima (Columbia) tidak mengirim NAK. Penerima tetap mengirim ACK dengan nomor yang sama, yaitu byte berikutnya yang masih ia tunggu, setiap kali segmen berikutnya tiba (duplicate ACK). Setelah 3 duplicate ACK berturut-turut, pengirim (Laterano) mengirim ulang segmen yang hilang tanpa menunggu timer habis (fast retransmit). Kalau tidak ada cukup dupACK, retransmisi baru terjadi setelah timeout. Akibatnya ada paket retransmisi, cwnd menurun, dan throughput serta waktu transfer lebih buruk daripada Skenario 1.

```sh
tc qdisc del dev eth1 root
tc qdisc del dev eth2 root
tc qdisc add dev eth1 root netem loss 1%
tc qdisc add dev eth2 root netem loss 1%
tc qdisc show dev eth1
tc qdisc show dev eth2
```

#### Soal B.2 - Tabel Observasi Retransmisi & Duplicate ACK

> Isi tabel berikut berdasarkan capture Wireshark dan output `curl`.

| No | Parameter/Aspek | Field Wireshark | Instruksi | Jawaban |
| --- | --- | --- | --- | --- |
| 1 | Persentase loss yang disimulasikan | `tc qdisc show` | Tunjukkan nilai setup packet loss (1%) pada `tc netem`. | `tc qdisc show dev eth1` dan `tc qdisc show dev eth2` pada RhodesIsland menampilkan `netem ... loss 1%`. |
| 2 | Prediksi vs aktual: reaksi penerima terhadap loss (Columbia) | `tcp.analysis.duplicate_ack` | Bandingkan prediksi dengan pola dupACK aktual (ACK bernomor sama berulang), bukan NAK. | Prediksi (B.1): penerima tidak mengirim NAK, melainkan duplicate ACK dengan nomor yang sama setiap kali segmen berikutnya tiba. <br> Aktual: pada capture Columbia ditemukan `<N>` paket duplicate ACK (tcp.analysis.duplicate_ack). Contoh rangkaian: semua paketnya punya Ack=(nomor yang sama), ditandai TCP Dup ACK #1, #2, #3. |
| 3 | Jumlah retransmisi | `tcp.analysis.retransmission` | Tunjukkan paket yang dikirim ulang akibat tidak menerima ACK yang diharapkan. | Jumlah retransmisi (tcp.analysis.retransmission) pada capture Laterano: `<N>` paket, terdiri dari `<F>` fast retransmission dan `<R>` retransmission biasa. Contoh: paket 319 (tcp.seq=`<seq>`, tcp.len=1448) ditandai [TCP Fast Retransmission], dikirim dari 10.182.4.15 ke 10.182.3.100. <br> Retransmisi terjadi karena Laterano tidak menerima ACK yang diharapkan untuk segmen tersebut: segmen data hilang di RhodesIsland akibat netem loss 1%, sehingga Columbia mengirim duplicate ACK dan Laterano mengirim ulang segmen yang hilang. |
| 4 | Verifikasi SEQ segmen retransmisi = SEQ segmen asli yang hilang | `tcp.seq`, `tcp.len` | Bandingkan paket `tcp.analysis.retransmission` dengan segmen aslinya, buktikan bahwa retransmisi mengirim ulang byte yang identik, bukan data baru. | Segmen asli: seq = 4145841992, len = 1514<br>Retransmisi: seq = 4145841992, len = 1514 |
| 5 | Jumlah duplicate ACK (Columbia) | `tcp.analysis.duplicate_ack` | Tunjukkan bahwa sedikitnya 3 dupACK berturut-turut memicu fast retransmit. | — |
| 6 | RTT sebelum dan sesudah loss | `tcp.analysis.ack_rtt` | Tunjukkan bahwa RTT cenderung meningkat karena waktu recovery. | — |
| 7 | Fast retransmit terdeteksi beserta selisih waktunya | `tcp.analysis.fast_retransmission`, `frame.time_relative` | Ukur selisih waktu antara dupACK ke-3 dan retransmisi yang menyusul, bandingkan dengan potensi timeout. | Fast retransmit terdeteksi: paket 319 (tcp.seq=4145841992), [TCP Fast Retransmission]. <br> dupACK: paket 266 (#1), 268 (#2), 270 (#3), semuanya Ack=4145841992. <br> dupACK ke-3: t = 158.757701 s <br> Retransmisi: t = 158.760038 s dengan Selisih = 2,34 ms. <br> Perbandingan: tanpa dupACK, pengirim harus menunggu RTO (minimal ±200 ms), sehingga fast retransmit ±85x lebih cepat dan link tidak menganggur lama. |
| 8 | Timeout retransmission | `frame.time_relative` | Terjadi bila tidak ada dupACK yang memicu fast retransmit. Jika tidak muncul, jelaskan apa yang diamati beserta alasannya. | Retransmisi non-fast hanya satu: paket 1139 (t = 158.805371 s), mengirim ulang segmen dari frame 1078 dengan jeda 5,902 ms. Jeda ini jauh di bawah RTO minimum Linux (±200 ms), sehingga bukan timeout. |
| 9 | Throughput rata-rata | `curl -w` (`speed_download`) | Bandingkan dengan throughput pada Skenario 1 (tanpa loss). | Throughput rata-rata (curl speed_download), tes5M.bin: <br> - Skenario 1 (tanpa loss): 20.351.925 B/s (≈ 162,8 Mbit/s) <br> - Skenario 2 (loss 1%)   : 5.351.531 B/s (≈ 42,8 Mbit/s) <br> Throughput turun ±73,7% (±3,8x lebih rendah). Setiap segmen yang hilang memicu retransmisi dan TCP menurunkan cwnd, sehingga laju kirim turun dan data yang berguna per detik berkurang. |
| 10 | Lama transfer total | `curl -w` (`time_total`) | Tunjukkan bahwa waktu transfer meningkat dibanding Skenario 1. | (Menggunakan output di nomor 9) <br> Lama transfer total (curl time_total), tes5M.bin: <br> - Tanpa loss : 0,245677 s <br> - Loss 1%    : 0,934312 s <br> Waktu transfer naik 0,688635 s (±3,8x lebih lama). Segmen yang hilang harus dikirim ulang dan cwnd menurun setelah loss, sehingga file selesai lebih lambat. |
| 11 | Pola congestion window (perkiraan) | `tcp.analysis.bytes_in_flight` | Tunjukkan bahwa cwnd menurun setelah loss, indikasi congestion control aktif. | — |

#### B.3 - Pertanyaan Analisis / Refleksi

> 1. Bagaimana TCP mengetahui bahwa sebuah segmen hilang?
>
> 2. Jelaskan perbedaan antara timeout retransmission dan fast retransmit.
>
> 3. Mengapa throughput menurun saat terjadi packet loss?
>
> 4. Bagaimana Wireshark menunjukkan bahwa TCP tetap reliable meskipun terjadi packet loss?

**Jawaban:**

**1. Bagaimana TCP tahu segmen hilang?**

TCP mengetahui sebuah segmen hilang lewat dua tanda. Pertama, duplicate ACK: ketika ada segmen yang hilang, penerima terus mengirim ACK dengan nomor yang sama karena masih menunggu byte yang belum datang, dan tiga duplicate ACK berturut-turut menandakan segmen itu hilang. Kedua, timeout: jika ACK yang ditunggu tidak datang sampai timer retransmisi (RTO) habis, pengirim menganggap segmen tersebut hilang.

**2. Perbedaan timeout retransmission dan fast retransmit**

Timeout retransmission terjadi ketika timer RTO habis tanpa ada ACK, sehingga pengirim harus menunggu cukup lama (minimal sekitar 200 ms di Linux) sebelum mengirim ulang, dan cwnd turun drastis lalu TCP memulai slow start lagi. Fast retransmit terjadi saat pengirim menerima tiga duplicate ACK berturut-turut, sehingga segmen yang hilang langsung dikirim ulang tanpa menunggu timer habis, dan penurunan cwnd-nya lebih ringan. Pada capture, duplicate ACK ke-3 (paket 270) muncul di 158,757701 s dan retransmisinya (paket 319) di 158,760038 s, selisih sekitar 2,34 ms, jauh lebih cepat daripada menunggu RTO.

**3. Mengapa throughput menurun saat loss?**

Throughput turun karena dua hal. Segmen yang hilang harus dikirim ulang sehingga sebagian bandwidth terpakai untuk data duplikat. Selain itu, TCP menganggap loss sebagai tanda congestion lalu mengecilkan cwnd, sehingga laju kirim turun dan baru naik pelan-pelan lagi. Hal ini terlihat pada hasil curl: speed turun dari 20.351.925 B/s menjadi 5.351.531 B/s (sekitar 73,7%), dan waktu transfer naik dari 0,245677 s menjadi 0,934312 s (sekitar 3,8 kali lebih lama).

**4. Bukti di Wireshark bahwa TCP tetap reliable**

Meskipun terjadi loss, data tetap sampai utuh. Segmen yang hilang dikirim ulang dengan tcp.seq dan tcp.len yang sama, sehingga byte yang dikirim identik dan bukan data baru. Penerima menandai celah dengan duplicate ACK bernomor sama (Ack=4145841992), lalu setelah retransmisi tiba, nomor ACK maju lagi melewati celah itu. Terakhir, curl melaporkan size=5000000B, sama persis dengan ukuran file, yang membuktikan tidak ada byte yang hilang.

---

# Bagian C - TCP Congestion Control (CUBIC vs Reno)

## a. Tujuan Pembelajaran

1. Mengamati mekanisme TCP congestion control pada kondisi jaringan dengan packet loss dan latency.
2. Membandingkan perilaku algoritma congestion control Cubic dan Reno melalui grafik TCP Stream pada Wireshark.
3. Membaca TCP Stream Graph secara tepat dan tidak menyamakan grafik dengan nilai cwnd yang diukur langsung.

## b. Tools yang Dibutuhkan

- `tc` (`netem` dan `tbf`) di RhodesIsland, untuk simulasi loss, latency, dan pembatasan rate sekaligus
- `sysctl`, untuk mengganti algoritma congestion control pada Laterano (sisi pengirim data)
- Wireshark dengan fitur Statistics -> TCP Stream Graph

## c. Problem C.3 Skenario 3: Loss, Latency, dan Perbandingan Algoritma Congestion Control

**Soal:**

Terapkan kondisi jaringan dengan packet loss 1%, latency 100 ms, dan pembatasan rate 2 Mbps pada eth1 dan eth2, lalu amati perilaku TCP congestion control Cubic dan Reno.

### Langkah Kerja

1. Di RhodesIsland, hapus qdisc sebelumnya, lalu terapkan kombinasi pembatasan rate (`tbf`) dan gangguan loss/latency (`netem`) secara berantai pada kedua interface:

```sh
$ tc qdisc del dev eth1 root
$ tc qdisc del dev eth2 root
$ tc qdisc add dev eth1 root handle 1: tbf rate 2mbit burst 32kbit latency 100ms
$ tc qdisc add dev eth1 parent 1:1 handle 10: netem delay 100ms loss 1%
$ tc qdisc add dev eth2 root handle 1: tbf rate 2mbit burst 32kbit latency 100ms
$ tc qdisc add dev eth2 parent 1:1 handle 10: netem delay 100ms loss 1%
$ tc qdisc show dev eth1
$ tc qdisc show dev eth2
```

2. Sebelum menjalankan pengujian, gambarkan dulu prediksi bentuk kurva congestion window setelah terjadi loss, untuk masing-masing algoritma: apakah cwnd akan turun tajam lalu naik linear (dugaan untuk Reno), atau turun lalu naik cepat mendekati titik sebelumnya (dugaan untuk Cubic)? Catat prediksi ini sebelum melihat grafik aktual.
3. Di Laterano (sisi pengirim data), atur algoritma congestion control ke cubic:

```sh
$ ip tcp_metrics flush
$ sysctl -w net.ipv4.tcp_no_metrics_save=1
$ sysctl -w net.ipv4.tcp_congestion_control=cubic
$ sysctl net.ipv4.tcp_congestion_control
```

4. Di antara Columbia dan Switch1, mulai capture pada interface eth0 (membuka Wireshark), terapkan filter tampilan:

```
tcp.port == 9999
```

5. Kemudian start capture menggunakan Wireshark di Laterano dengan cara yang sama seperti di Columbia (sama seperti Skenario 1 dan 2).
6. Lakukan download berkas `tes5M.bin` sambil mengamati aktivitas Wireshark:

```sh
$ curl -o /dev/null -s -w "size=%{size_download}B speed=%{speed_download}B/s total=%{time_total}s\n" http://10.182.4.15:9999/tes5M.bin
```

7. Hentikan capture Wireshark di Columbia setelah download selesai, lalu telusuri paket-paket yang tercatat (hasil capture dari port eth Columbia dan Laterano).
8. Setelah capture selesai, gunakan menu Statistics -> TCP Stream Graph pada Wireshark untuk membaca pola congestion window dan RTT:
9. **Time - Sequence (Stevens):** menunjukkan pertumbuhan congestion window dari waktu ke waktu.
10. **Round Trip Time Graph:** menunjukkan stabilitas RTT sepanjang transfer.
11. **Throughput Graph:** menunjukkan fluktuasi laju data yang berhasil dikirim.
12. Ulangi seluruh langkah di atas dengan mengganti algoritma congestion control menjadi reno, lalu bandingkan hasil kedua grafik:

```sh
$ sysctl -w net.ipv4.tcp_congestion_control=reno
$ ip tcp_metrics flush
$ sysctl net.ipv4.tcp_congestion_control
```

13. Cocokkan bentuk kurva cwnd aktual pada grafik Time - Sequence dengan prediksi yang sudah dituliskan sebelumnya, jelaskan bila bentuknya berbeda dari dugaan.

> [!NOTE]
> Skenario 3: loss 1%, delay 100 ms, rate limit 2 Mbps (tbf + netem) pada eth1 dan eth2 RhodesIsland. Algoritma diatur di **Laterano** (sender). Download `tes5M.bin`. Untuk parameter 4, 5, dan 9 gunakan capture **Columbia**; selain itu gunakan capture **Laterano**. Jangan klaim mengukur cwnd secara langsung dari grafik Time-Sequence / `bytes_in_flight`.

#### Soal C.1 - Setup Gangguan & Prediksi Kurva cwnd

> Tunjukkan konfigurasi `tc` (tbf + netem). Tuliskan prediksi bentuk kurva cwnd untuk Reno dan CUBIC sebelum melihat grafik.

**Jawaban:**

Prediksi RENO: cwnd turun tajam (sekitar separuh) setelah loss, lalu naik linear pelan-pelan, bentuknya gigi gergaji.

Prediksi CUBIC: cwnd turun setelah loss, lalu naik cepat mendekati titik sebelum loss, jadi pemulihannya lebih cepat dan throughput lebih tinggi daripada Reno.

Konfigurasi `tc` yang diterapkan di RhodesIsland:

```sh
tc qdisc del dev eth1 root
tc qdisc del dev eth2 root
tc qdisc add dev eth1 root handle 1: tbf rate 2mbit burst 32kbit latency 100ms
tc qdisc add dev eth1 parent 1:1 handle 10: netem delay 100ms loss 1%
tc qdisc add dev eth2 root handle 1: tbf rate 2mbit burst 32kbit latency 100ms
tc qdisc add dev eth2 parent 1:1 handle 10: netem delay 100ms loss 1%
tc qdisc show dev eth1
tc qdisc show dev eth2
```

#### Soal C.2 - Konfigurasi Algoritma Congestion Control

> Atur algoritma di Laterano (sender) lalu jalankan download `tes5M.bin` dari Columbia untuk masing-masing algoritma.

| Langkah | CUBIC | Reno |
| --- | --- | --- |
| Command di Laterano | `ip tcp_metrics flush`<br>`sysctl -w net.ipv4.tcp_no_metrics_save=1`<br>`sysctl -w net.ipv4.tcp_congestion_control=cubic`<br>`sysctl net.ipv4.tcp_congestion_control` | `sysctl -w net.ipv4.tcp_congestion_control=reno`<br>`ip tcp_metrics flush`<br>`sysctl net.ipv4.tcp_congestion_control` |
| Command di Columbia | `curl ... tes5M.bin` (sama seperti Skenario 1 dan 2) | `curl ... tes5M.bin` (sama seperti CUBIC) |

#### Soal C.3 - Tabel Perbandingan CUBIC vs Reno

> Isi tabel berikut dengan hasil aktual untuk CUBIC dan Reno (nilai + screenshot), lalu tulis analisis singkat pada kolom catatan.
> Setiap sel kosong untuk CUBIC dan RENO pada tabel di bawah **wajib** berisi **nilai aktual** dan **screenshot** sebagai bukti, kecuali parameter nomor 1.

| No | Parameter/Aspek | Field Wireshark / Command | CUBIC | Reno | Catatan analisis |
| --- | --- | --- | --- | --- | --- |
| 1 | Prediksi vs aktual: bentuk kurva cwnd | Time - Sequence Graph | — | — | Prediksi awal cocok dengan grafik aktual. CUBIC memulihkan cwnd jauh lebih cepat via fungsi kubik pada jaringan ber-RTT tinggi, sedangkan Reno terbatas oleh pertumbuhan linier (1 MSS/RTT) yang terhambat oleh delay 100 ms. |
| 2 | RTT rata-rata (ms) | `tcp.analysis.ack_rtt` | — | — | RTT stabil menunjukkan congestion control yang efisien. |
| 3 | RTT maksimum (ms) | `tcp.analysis.ack_rtt` | — | — | RTT tinggi mengindikasikan queue buildup pada jalur. |
| 4 | Throughput (B/s) (Columbia) | `curl -w` (`speed_download`) | Speed: 124277B/s | Speed: 78157B/s | CUBIC menghasilkan throughput sekitar 59% lebih tinggi dibandingkan Reno pada jaringan ini. Hal ini terjadi karena CUBIC menggunakan fungsi kubik berbasis waktu (bukan jumlah RTT) untuk menaikkan window pengiriman. Setelah terjadi packet loss, CUBIC mampu menaikkan laju pengiriman (cwnd) jauh lebih cepat menuju kapasitas maksimum link dibanding Reno yang menaikkan laju secara perlahan dan linier (1 MSS per RTT). |
| 5 | Waktu transfer total (s) (Columbia) | `curl -w` (`time_total`) | Total = 40.232400s | Total = 63.973305s | CUBIC menyelesaikan pengunduhan berkas 5 MB sekitar 23,7 detik lebih cepat (~37% lebih singkat) daripada Reno. Karena jaringan memiliki delay tinggi (100 ms) dan pembatasan laju (2 Mbps) beserta packet loss 1%, pemulihan window Reno yang lambat menyebabkan durasi transfer membengkak, sedangkan pemulihan agresif CUBIC memaksimalkan pemanfaatan bandwidth yang tersedia. |
| 6 | Jumlah retransmisi | `tcp.analysis.retransmission` | Filter: `tcp.analysis.retransmission` (jumlah pada status bar Wireshark: displayed) | Filter: `tcp.analysis.retransmission` (jumlah pada status bar Wireshark: displayed) | Retransmisi yang banyak mengindikasikan algoritma bersifat loss-based. |
| 7 | Jumlah duplicate ACK | `tcp.analysis.duplicate_ack` | Filter: `tcp.analysis.duplicate_ack` (jumlah pada status bar Wireshark: displayed) | Filter: `tcp.analysis.duplicate_ack` (jumlah pada status bar Wireshark: displayed) | Indikasi segmen yang sempat hilang di tengah transfer. |
| 8 | Pola congestion window (awal - akhir) | `tcp.analysis.bytes_in_flight` | — | — | CUBIC: Kurva naik melengkung/cepat setelah loss (fungsi kubik). <br> RENO: Kurva naik linear/miring konstan setelah loss (gigi gergaji). |
| 9 | Nilai receive window rata-rata (Columbia) | `tcp.window_size_value` | — | — | Menunjukkan kapasitas buffer di sisi penerima. |
| 10 | Stabilitas throughput (grafik) | Throughput Graph | — | — | CUBIC: Cenderung lebih stabil dan tinggi setelah fase recovery. <br> RENO: Lebih sering turun-naik tajam (fluktuatif) karena pertumbuhan cwnd yang lambat. |

#### C.3 - Pertanyaan Analisis / Refleksi

> 1. Algoritma mana yang memberikan throughput lebih tinggi pada kondisi jaringan ini? Mengapa demikian?
>
> 2. Apakah ada hubungan antara jumlah retransmisi dan jenis algoritma congestion control yang digunakan?
>
> 3. Apa perbedaan bentuk kurva congestion window antara Reno dan Cubic pada grafik Time-Sequence (Stevens)?
>
> 4. Algoritma mana yang paling sensitif terhadap RTT tinggi?
>
> 5. Algoritma mana yang lebih efisien pada jaringan dengan latency tinggi seperti pada skenario ini? Jelaskan.

**Jawaban:**

1. Algoritma CUBIC memberikan throughput lebih tinggi. Hal ini terjadi karena CUBIC menggunakan fungsi kubik berbasis waktu untuk menaikkan window pengiriman, sehingga pemulihannya setelah packet loss jauh lebih cepat dibandingkan Reno yang hanya menaikkan laju secara linear (1 MSS per RTT) dan terhambat oleh delay 100 ms.

2. Ya, ada hubungan. Meskipun keduanya loss-based, CUBIC cenderung lebih agresif dalam mengisi bandwidth, yang bisa menyebabkan antrian (queue buildup) dan memicu sedikit lebih banyak retransmisi atau duplicate ACK dibandingkan Reno yang lebih konservatif dalam menaikkan cwnd.

3. Pada grafik Time-Sequence, kurva Reno menunjukkan pola "gigi gergaji" yang khas: turun tajam saat loss, lalu naik secara linear (garis lurus) yang lambat. Sebaliknya, kurva CUBIC menunjukkan penurunan saat loss, tetapi naik kembali dengan kurva yang lebih melengkung dan curam (fungsi kubik), mencapai titik sebelum loss lebih cepat.

4. Algoritma Reno paling sensitif terhadap RTT tinggi. Karena kenaikannya bergantung pada jumlah RTT (1 MSS per RTT), maka RTT yang besar (100 ms) akan membuat proses pemulihan congestion window menjadi sangat lambat dibandingkan CUBIC yang bergantung pada waktu absolut.

5. Algoritma CUBIC lebih efisien. Jaringan dengan latency tinggi (100 ms) dan packet loss membutuhkan algoritma yang bisa memanfaatkan bandwidth yang tersedia dengan cepat setelah gangguan berlalu. CUBIC melakukannya melalui fungsi kubiknya, sehingga waktu transfer total lebih singkat dan throughput lebih tinggi dibandingkan pendekatan linear Reno.

---

# Bagian D - Verifikasi Perilaku TCP vs UDP

## a. Tujuan Pembelajaran

1. Membandingkan reaksi protokol TCP dan UDP terhadap kondisi jaringan yang identik (packet loss).
2. Menegaskan kembali konsep reliable vs unreliable transport melalui bukti empiris, bukan sekadar definisi teoritis.
3. Menginterpretasikan bitrate pengirim, throughput penerima, dan datagram loss di `iperf3`.

## b. Tools yang Dibutuhkan

- `iperf3`, dijalankan dalam mode TCP (default) maupun mode UDP (opsi `-u`)
- `tc netem` di RhodesIsland, dengan konfigurasi loss yang sama untuk kedua pengujian

## c. Problem D.1 - D.3 Uji Banding TCP vs UDP pada Kondisi Loss yang Sama

**Soal:**

Terapkan packet loss 5% pada eth1 dan eth2 RhodesIsland, lalu bandingkan hasil pengujian `iperf3` dalam mode TCP dan mode UDP pada kondisi tersebut.

### Langkah Kerja

1. Di RhodesIsland, hapus qdisc sebelumnya, lalu terapkan packet loss 5% (nilai yang lebih besar dari Bagian B agar efeknya lebih kontras) pada kedua interface:

```sh
$ tc qdisc del dev eth1 root
$ tc qdisc del dev eth2 root
$ tc qdisc add dev eth1 root netem loss 5%
$ tc qdisc add dev eth2 root netem loss 5%
$ tc qdisc show dev eth1
$ tc qdisc show dev eth2
```

2. Di Laterano, jalankan `iperf3` sebagai server:

```sh
$ iperf3 -s
```

3. Di Columbia, jalankan pengujian mode TCP (default) terlebih dahulu:

```sh
$ iperf3 -c 10.182.4.15 -t 10
```

4. Catat throughput yang dilaporkan, lalu jalankan pengujian mode UDP dengan target bitrate tetap 5 Mbps:

```sh
$ iperf3 -c 10.182.4.15 -u -b 5M -t 10
```

5. Bandingkan kedua laporan akhir `iperf3`, perhatikan baris Lost/Total Datagrams pada hasil UDP, yang tidak muncul pada hasil TCP.

> [!NOTE]
> Packet loss 5% pada eth1 dan eth2 RhodesIsland. `iperf3 -s` di Laterano; client di Columbia.

#### Soal D.1 - Setup Gangguan (`tc qdisc show`)

> Loss 5% diterapkan pada kedua interface RhodesIsland.

**Jawaban:**

```sh
tc qdisc del dev eth1 root
tc qdisc del dev eth2 root
tc qdisc add dev eth1 root netem loss 5%
tc qdisc add dev eth2 root netem loss 5%
tc qdisc show dev eth1
tc qdisc show dev eth2
```

#### Soal D.2 - Tabel Hasil Pengujian TCP vs UDP

> Jalankan pengujian TCP lalu UDP, catat hasil sender/receiver (nilai + screenshot), lalu bandingkan. Kesimpulan harus didasarkan pada hasil pengukuran yang setara.
> Setiap sel kosong pada tabel di bawah **wajib** berisi **nilai aktual** dan **screenshot** sebagai bukti.

| Parameter | TCP | UDP |
| --- | --- | --- |
| Command di Columbia | `iperf3 -c 10.182.4.15 -t 10` | `iperf3 -c 10.182.4.15 -u -b 5M -t 10` |
| Bitrate/throughput sender | 7.86 Mbits/sec (Dari baris summary sender) | 5.00 Mbits/sec (Sesuai target -b 5M) |
| Bitrate/throughput receiver | 7.76 Mbits/sec (Sedikit lebih rendah karena overhead retransmisi) | 4.75 Mbits/sec (Turun karena paket hilang di jaringan) |
| Retransmission yang dilaporkan | 367 (Kolom Retr pada summary sender) | Tidak ada retransmission bawaan UDP |
| Lost / Total Datagrams | Tidak tersedia dalam format TCP | 217 / 4317 (5%) (Persentase loss sesuai simulasi netem) |
| Jitter (jika tercantum) | — | 0.130 ms (Variasi delay antar paket sangat kecil) |
| Ringkasan respons terhadap loss | Throughput turun drastis dari kapasitas link asli menjadi ~7.8 Mbps. TCP mendeteksi loss sebagai congestion, melakukan retransmisi masif (367 kali), dan mengecilkan congestion window secara agresif sehingga laju kirim tidak stabil. | Sender tetap memaksa mengirim tepat 5 Mbps tanpa peduli loss. Receiver hanya menerima 4.75 Mbps (kehilangan 5% datagram). Tidak ada mekanisme recovery; data yang hilang permanen hilang. |

#### D.3 - Pertanyaan Analisis / Refleksi

> 1. Mengapa hasil iperf3 mode UDP menampilkan persentase datagram loss, sedangkan hasil mode TCP tidak?
>
> 2. Mengapa throughput UDP tetap mendekati nilai yang diminta (`-b`) meskipun terjadi packet loss, sedangkan throughput TCP menurun?
>
> 3. Mengapa throughput TCP bisa turun pada kondisi loss, sedangkan UDP tetap mencoba mengirim sesuai target bitrate?
>
> 4. Untuk jenis aplikasi seperti apa pilihan UDP lebih tepat dibanding TCP, berdasarkan hasil pengamatan ini?

**Jawaban:**

1. Karena UDP bersifat connectionless dan unreliable, ia tidak memiliki mekanisme ACK atau retransmisi. Oleh karena itu, setiap paket yang dibuang oleh netem akan tercatat eksplisit sebagai "Lost" di sisi receiver. Sebaliknya, TCP menyembunyikan loss tersebut melalui mekanisme retransmisi otomatis. Paket yang hilang dikirim ulang hingga sampai, sehingga di level aplikasi/receiver semua data dianggap utuh dan tidak ada laporan "loss". Loss pada TCP hanya terlihat secara implisit melalui kolom Retr (retransmissions) dan penurunan throughput.

2. UDP tidak memiliki congestion control. Ia hanya mengikuti instruksi bitrate yang diberikan via flag -b (5 Mbps) dan terus menembakkan paket dengan laju tersebut terlepas dari kondisi jaringan. TCP, sebaliknya, menganggap packet loss sebagai tanda kemacetan jaringan (congestion). Saat mendeteksi loss, TCP langsung menurunkan congestion window (cwnd) dan memperlambat laju pengiriman untuk mencegah memperparah kemacetan, sehingga throughput efektifnya turun jauh di bawah kapasitas fisik link.

3. Penurunan throughput TCP adalah fitur desain untuk keandalan (reliability) dan keadilan (fairness). Dengan mengurangi laju saat loss terdeteksi, TCP memastikan jaringan tidak kolaps total dan memberikan kesempatan bagi aliran data lain. UDP dirancang untuk kecepatan dan real-time, bukan keandalan. Ia mengorbankan integritas data demi mempertahankan laju konstan, karena bagi aplikasi seperti streaming video atau VoIP, keterlambatan akibat retransmisi seringkali lebih buruk daripada kehilangan beberapa frame data.

4. UDP lebih tepat untuk aplikasi real-time yang toleran terhadap loss, seperti:
   - Video Streaming / Live Broadcast: Kehilangan beberapa frame tidak merusak pengalaman menonton, tapi buffering akibat retransmisi TCP sangat mengganggu.
   - VoIP / Video Call: Delay dan jitter harus diminimalkan. Suara yang sedikit putus-putus (akibat loss 5%) masih lebih baik daripada suara yang datang terlambat 2 detik karena menunggu retransmisi.
   - Online Gaming (FPS/MOBA): Posisi pemain harus diperbarui secepat mungkin. Data posisi lama yang baru saja sampai karena retransmisi sudah tidak relevan dan justru menyebabkan rubber-banding.
   - DNS Query: Permintaan kecil yang butuh respon instan; jika timeout, client cukup retry sendiri tanpa beban overhead handshake TCP.

#### Tabel Perbandingan Hasil - Materi Sumber

| Aspek | Hasil pada TCP | Hasil pada UDP |
| --- | --- | --- |
| Throughput terukur | Cenderung di bawah kapasitas link, menurun mengikuti tingkat loss | Mendekati bitrate yang diminta (`-b`), relatif tidak terpengaruh loss |
| Retransmisi | Tercatat oleh iperf3 (kolom Retr) | Tidak ada mekanisme retransmisi sama sekali |
| Packet/datagram loss | Tidak dilaporkan langsung, loss "tersembunyi" karena sudah ditangani lewat retransmisi | Dilaporkan eksplisit sebagai persentase Lost/Total Datagrams |
| Reaksi terhadap loss | Congestion window menurun, laju pengiriman menyesuaikan diri (self-throttling) | Tidak ada reaksi apa pun, pengirim tetap pada bitrate yang diminta |
| Jaminan data sampai utuh | Ya, seluruh byte dijamin sampai dan berurutan | Tidak, datagram yang hilang tidak pernah dikirim ulang |

---

# Bagian E - Tugas Mandiri

> [!IMPORTANT]
> Gunakan topologi dari pengerjaan Bagian A - D sebelumnya. Isi tabel dengan **hasil aktual**, bukan angka perkiraan.

### Yang Perlu Dilakukan

1. Tuliskan konfigurasi IP/network dan konfigurasi service tiap node sesuai yang ada di topologi dan tabel pemetaan node.
2. Kerjakan Skenario 1 (Bagian A.4), 2 (Bagian B.3) dan 3 (Bagian C.3) secara lengkap.
3. Berikan screenshot command/pengerjaan untuk masing-masing parameter/aspek pada tabel.
4. Ulangi Skenario 1 (Bagian A, transfer tanpa gangguan) untuk ketiga ukuran berkas uji: `tes1000.bin`, `tes10000.bin`, dan `tes5M.bin`. Kemudian lengkapi tabel berikut berdasarkan hasil capture Wireshark di Laterano dan Columbia serta output `curl`, lalu berikan kesimpulan mengenai pengaruh ukuran berkas terhadap parameter-parameter TCP yang diamati.
5. Lengkapi tabel perbandingan berikut berdasarkan hasil aktual; jangan mengisi angka perkiraan.
6. Tuliskan kesimpulan pengaruh ukuran berkas terhadap jumlah segmen, RTT, throughput, waktu transfer, dan indikator reliability.

#### Soal E.1 - Ulangi Skenario 1 untuk Tiga Ukuran Berkas

> Ulangi Skenario 1 (tanpa gangguan) untuk `tes1000.bin`, `tes10000.bin`, dan `tes5M.bin`, lalu lengkapi tabel berikut berdasarkan capture Wireshark di Laterano dan Columbia serta output `curl`.
> Setiap sel kosong pada tabel di bawah **wajib** berisi **nilai aktual** dan **screenshot** sebagai bukti.

**Tabel Observasi Kondisi Normal**

| No | Parameter/Aspek | Field Wireshark | tes1000.bin | tes10000.bin | tes5M.bin |
| --- | --- | --- | --- | --- | --- |
| 1 | Ukuran berkas (B) | - | 1.000 B | 10.000 B | 5.000.000 B |
| 2 | Jumlah total segmen TCP | Statistics -> Conversations -> TCP | — | — | — |
| 3 | RTT rata-rata (Laterano) | `tcp.analysis.ack_rtt` | — | — | — |
| 4 | Throughput (curl) | `speed_download` | 360620 B/s | 2744237 B/s | 58912244 B/s |
| 5 | Waktu total transfer (curl) | `time_total` | 0.002773 s | 0.003644 s | 0.084872 s |
| 6 | Sequence number awal client | `tcp.seq` (SYN) | — | — | — |
| 7 | Sequence number awal server | `tcp.seq` (SYN-ACK) | 1037109012 | 529632909 | 85409173 |
| 8 | Jumlah ACK yang dikirim client | Statistics -> Endpoints -> TCP | 4 | 9 | 1.051 |
| 9 | Status retransmisi (Laterano) | `tcp.analysis.retransmission` | 0 paket, membuktikan transfer berjalan reliable tanpa loss di kondisi normal. | 0 paket, membuktikan transfer berjalan reliable tanpa loss di kondisi normal. | 14 paket yang diretransmit. |

#### E.2 - Pertanyaan Kesimpulan

> 1. Apakah jumlah segmen TCP, RTT, dan throughput berubah secara proporsional terhadap ukuran berkas?
>
> 2. Parameter mana yang relatif tetap tidak bergantung pada ukuran berkas, dan mengapa demikian?
>
> 3. Apa perbedaan antara total waktu download yang diukur curl dengan RTT paket TCP?

**Jawaban:**

1. Jumlah segmen TCP berubah secara proporsional linear terhadap ukuran berkas. Karena MSS (Maximum Segment Size) bersifat konstan (biasanya ~1460 byte), maka file yang berukuran 5x lipat akan menghasilkan jumlah segmen data yang juga sekitar 5x lipat lebih banyak. Sebaliknya, RTT rata-rata dan Throughput tidak berubah secara proporsional. Pada kondisi normal tanpa gangguan jaringan, nilai RTT ditentukan oleh base latency fisik link dan pemrosesan router, bukan oleh besarnya data yang dikirim.

2. Parameter yang tetap konstan meliputi:
   - RTT Rata-rata & Maksimum: Ditentukan oleh jarak hop, delay propagasi, dan waktu pemrosesan antrean di router RhodesIsland/Lungmen/Ursus. Ukuran payload tidak mempengaruhi waktu tempuh satu paket ACK.
   - Initial Sequence Number (ISN): Dihasilkan secara acak oleh kernel OS saat handshake, sama sekali tidak berkorelasi dengan ukuran file yang akan ditransfer.
   - Receive Window (RWIN): Merupakan kapasitas buffer memori di sisi penerima (Columbia). Nilainya ditetapkan saat koneksi dibangun dan hanya berubah jika terjadi tekanan memori atau congestion control, bukan karena ukuran file download.
   - Status Retransmisi: Dalam kondisi normal (Skenario 1), retransmisi harus selalu nol untuk semua ukuran file karena tidak ada packet loss atau error link.

3. Waktu download (time_total dari curl) adalah metrik makro/aplikasi yang mengukur durasi keseluruhan siklus hidup transfer: mulai dari resolusi DNS, three-way handshake, pengiriman seluruh segmen data, hingga four-way handshake penutupan koneksi. Nilai ini sangat bergantung pada ukuran file dan throughput efektif. Sementara RTT paket TCP adalah metrik mikro/protokol yang hanya mengukur waktu tempuh bolak-balik (round-trip) untuk satu segmen spesifik (dari pengiriman data hingga penerimaan ACK-nya). RTT merepresentasikan kesehatan dan latensi jalur jaringan pada level transport layer, sedangkan waktu curl merepresentasikan performa penyelesaian tugas transfer file pada level aplikasi.

---

## Ketentuan Bukti Pengerjaan

1. Sertakan tangkapan layar topologi GNS3 dan hasil uji konektivitas dari bagian Pre.
2. Sertakan command konfigurasi penting dan tabel route DHCP yang relevan, bukan hanya hasil ping.
3. Sertakan capture Wireshark yang memperlihatkan field atau filter yang dianalisis; satu tangkapan layar boleh menjadi bukti beberapa parameter bila terbaca jelas.
4. Pisahkan hasil pengujian normal, loss 1%, CUBIC, Reno, TCP, dan UDP agar data tidak tertukar.
5. Jika indikator bersifat probabilistik tidak terlihat (misalnya fast retransmit), jelaskan hasil yang benar-benar diamati beserta alasan yang masuk akal.
