# Topologi
Versi cisco: 9.0.1.0858

### Konfigurasi R Pusat

Sisi R-PUSAT memakai alamat host pertama (.1) di tiap link /30, dan yang lain .2  

R-PUSAT butuh 8 interface fisik (ISP, L1–L5, dan 2 LAN) plus tunnel virtual yang tidak membutuhkan port fisik, jadi total ada 8 port.

Lalu lintas Balai Kota ke Pelabuhan harus lewat tunnel, jadi route Pelabuhan di R-PUSAT nanti harus diarahkan ke 192.168.14.2 [ interface pusat ], bukan ke 10.113.66.2 [ interface L2 ]   

### Alasan Penggunaan Satu LAN, Satu Switch, dan Satu Port Router

Dalam perancangan jaringan, setiap LAN dihubungkan ke satu switch dan satu port router secara terpisah untuk mempermudah pengelolaan jaringan, meningkatkan keamanan, serta menjaga kestabilan komunikasi data. Berikut beberapa alasan penerapannya:

1. Pemisahan Jaringan: Setiap LAN memiliki subnet dan gateway sendiri sehingga lalu lintas broadcast, seperti ARP dan DHCP Discover, tetap berada dalam jaringan masing-masing. Hal ini penting terutama pada jaringan CCTV yang memiliki banyak perangkat.

2. Isolasi Gangguan: Pemisahan switch memungkinkan gangguan pada satu LAN tidak langsung memengaruhi LAN lainnya. Misalnya, ketika jaringan CCTV mengalami gangguan, layanan Server DC, Puskesmas, dan Balai Kota tetap dapat beroperasi.

3. Keamanan Jaringan: Setiap LAN memiliki batas jaringan yang jelas sehingga akses antarjaringan dapat dikendalikan menggunakan ACL pada interface router. Dengan demikian, jaringan WiFi Publik dapat dipisahkan dari sistem penting seperti server dan SCADA Pelabuhan.

4. Kemudahan Routing: Setiap port router terhubung langsung dengan satu subnet sehingga konfigurasi routing lebih terstruktur. Pembagian ini juga memudahkan penerapan CIDR dan peringkasan rute (route summarization) berdasarkan wilayah.

5. Pembagian Bandwidth: Setiap LAN memiliki jalur koneksi sendiri menuju router sehingga lalu lintas data antarlayanan tidak saling membebani pada jalur uplink yang sama. Contohnya, trafik CCTV tidak menggunakan uplink LAN yang diperuntukkan bagi layanan lainnya.

6. Pengelolaan DHCP: Setiap LAN dapat menggunakan kelompok alamat IP (DHCP pool) yang berbeda sesuai kebutuhan. Jika layanan DHCP berada di Server DC, router dapat menggunakan `ip helper-address` untuk meneruskan permintaan DHCP dari LAN terkait.

Dengan pendekatan ini, jaringan menjadi lebih terstruktur, mudah dikelola, aman, dan lebih mudah ditangani ketika terjadi gangguan, terutama pada infrastruktur yang melayani berbagai kebutuhan dalam satu wilayah.

### Jumlah Port Fisik  

| Router | Port fisik | + L7 saat OSPF |
| :--- | :---: | :---: |
| R-PUSAT | 8 | 8 |
| R-UTARA | 4 (L2, L6, 2 LAN) | 5 |
| R-DC | 3 (L1, 2 LAN) | 4 |
| R-BARAT, R-TIMUR, R-SELATAN, R-PELABUHAN | 3 | 3 |
| R-ISP | 2 | 2 |

Jenis Router : 2811 dengan module tambahan NM-2FE2W, WIC-1ENET  

### Alamat R-PUSAT

| Tujuan | Alamat R-PUSAT | Mask |
| :--- | :--- | :--- |
| R-ISP | `203.0.113.114` (ISP = `.113`, bebas ditukar, asal konsisten) | `/30` |
| L1 $\rightarrow$ R-DC | `10.112.97.1` | `/30` |
| L2 $\rightarrow$ R-UTARA | `10.113.66.1` | `/30` |
| L3 $\rightarrow$ R-SELATAN | `10.112.224.129` | `/30` |
| L4 $\rightarrow$ R-TIMUR | `10.112.164.1` | `/30` |
| L5 $\rightarrow$ R-BARAT | `10.112.33.1` | `/30` |
| LAN Pegawai Balai Kota | `10.113.192.1` | `/21 (255.255.248.0)` |
| LAN Server Balai Kota | `10.113.200.1` | `/27 (255.255.255.224)` |
| Tunnel GRE | `192.168.14.1` | `/30` |

### Static Route R-PUSAT

| Tujuan | Next hop | Lewat |
| :--- | :--- | :--- |
| `0.0.0.0/0` | `203.0.113.113` | R-ISP |
| `10.112.0.0/18` (Barat) | `10.112.33.2` | L5 |
| `10.112.64.0/18` (DC) | `10.112.97.2` | L1 |
| `10.112.128.0/18` (Timur) | `10.112.164.2` | L4 |
| `10.112.192.0/18` (Selatan) | `10.112.224.130` | L3 |
| `10.113.0.0/17` (Utara) | `10.113.66.2` | L2 |
| `10.113.128.0/18` (Pelabuhan) | `10.113.66.2` | L2, lewat R-UTARA |  

Selain server & mobile dijadikan PC

### Access Point ( AP-PT ) ke smartphone
Di perangkat AP: 
config > Port 1 > SSID ganti Wifi-Barat/Wifi-Utara/Wifi-Timur/Wifi-Selatan  
pakai WPA2PSK > pass phrasenya pwbarat/pwutara/pwtimur/pwselatan  

Di HP juga demikian: 
HP → Config → Wireless0
isi SSID dan passphrase yang sama dengan AP wilayahnya  

# Step by step

## 1. Server internet - R-ISP - R-PUSAT
<br>
R-ISP Fa0/0 ke R-PUSAT Fa0/0  
R-ISP Fa0/1 ke Server Internet Fa0  

Klik server, lalu Desktop > IP Configuration > Static:  
IP Address: 8.8.8.8  
Subnet Mask: 255.255.255.0  
Default Gateway: 8.8.8.1  
<br>
Lalu di CLI R-ISP
```
enable
configure terminal

! Bersihkan Fa0/0 dari IP server
interface FastEthernet0/0
 no ip address
 no description
 shutdown
exit

! Fa0/1 = Server Internet
interface FastEthernet0/1
 description KE-SERVER-INTERNET
 ip address 8.8.8.1 255.255.255.0
 no shutdown
exit

! Fa0/0 = R-PUSAT
interface FastEthernet0/0
 description KE-R-PUSAT
 ip address 203.0.113.113 255.255.255.252
 no shutdown
exit

! Route balik ke seluruh jaringan Smart City (/15)
!ip route 10.112.0.0 255.254.0.0 203.0.113.114

! wait, point soal 5 katanya no ip route
no ip route 10.112.0.0 255.254.0.0 203.0.113.114

end
write memory
```
<br>
Sekarang di R-PUSAT, untuk koneksi R-PUSAT ke R-ISP  
<br>
Verifikasi
- R-ISP: show ip interface brief. Fa0/0 dan Fa0/1 harus up/up  
- R-ISP: ping 8.8.8.8 dan ping 203.0.113.114  
- R-PUSAT: ping 203.0.113.113, lalu ping 8.8.8.8  
  
## 2. R-PUSAT ke host, R-PUSAT ke interface lainnya

PUSAT Fa0/1 ke R-UTARA Fa0/0  
R-PUSAT Fa1/0 ke SW-P_Pegawai-Balai Fa0/1  
R-PUSAT Fa1/1 ke SW-P_Server-Balai Fa0/1  
R-PUSAT Eth0/0/0 ke R-DC Eth0/0/0  
R-PUSAT Eth0/1/0 ke R-BARAT Eth0/0/0  
R-PUSAT Eth0/2/0 ke R-TIMUR Eth0/0/0  
R-PUSAT Eth0/3/0 ke R-SELATAN Eth0/0/0  
<br>

**R-PUSAT**
```
enable
configure terminal

interface FastEthernet0/1
 description L2-KE-R-UTARA
 ip address 10.113.66.1 255.255.255.252
 no shutdown
exit

interface FastEthernet1/0
 description LAN-PEGAWAI-BALAI-KOTA
 ip address 10.113.192.1 255.255.248.0
 no shutdown
exit

interface FastEthernet1/1
 description LAN-SERVER-BALAI-KOTA
 ip address 10.113.200.1 255.255.255.224
 no shutdown
exit

interface Ethernet0/0/0
 description L1-KE-R-DC
 ip address 10.112.97.1 255.255.255.252
 no shutdown
exit

interface Ethernet0/1/0
 description L5-KE-R-BARAT
 ip address 10.112.33.1 255.255.255.252
 no shutdown
exit

interface Ethernet0/2/0
 description L4-KE-R-TIMUR
 ip address 10.112.164.1 255.255.255.252
 no shutdown
exit

interface Ethernet0/3/0
 description L3-KE-R-SELATAN
 ip address 10.112.224.129 255.255.255.252
 no shutdown
exit

! Static route sesuai dokumen
ip route 10.112.0.0 255.255.192.0 10.112.33.2
ip route 10.112.64.0 255.255.192.0 10.112.97.2
ip route 10.112.128.0 255.255.192.0 10.112.164.2
ip route 10.112.192.0 255.255.192.0 10.112.224.130
ip route 10.113.0.0 255.255.128.0 10.113.66.2
ip route 10.113.128.0 255.255.192.0 10.113.66.2

end
write memory
```

**R-UTARA**
```
enable
configure terminal
hostname R-UTARA
no ip domain-lookup
interface FastEthernet0/0
 description L2-KE-R-PUSAT
 ip address 10.113.66.2 255.255.255.252
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.113.66.1
end
write memory
```
  
**R-DC**
```
enable
configure terminal
hostname R-DC
no ip domain-lookup
interface Ethernet0/0/0
 description L1-KE-R-PUSAT
 ip address 10.112.97.2 255.255.255.252
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.112.97.1
end
write memory
```
  
**R-BARAT**
```
enable
configure terminal
hostname R-BARAT
no ip domain-lookup
interface Ethernet0/0/0
 description L5-KE-R-PUSAT
 ip address 10.112.33.2 255.255.255.252
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.112.33.1
end
write memory
```
  
**R-TIMUR**
```
enable
configure terminal
hostname R-TIMUR
no ip domain-lookup
interface Ethernet0/0/0
 description L4-KE-R-PUSAT
 ip address 10.112.164.2 255.255.255.252
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.112.164.1
end
write memory
```
  
**R-SELATAN**
```
enable
configure terminal
hostname R-SELATAN
no ip domain-lookup
interface Ethernet0/0/0
 description L3-KE-R-PUSAT
 ip address 10.112.224.130 255.255.255.252
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.112.224.129
end
write memory
```
  
Verifikasi (di R-PUSAT)
show ip interface brief  
ping 10.113.66.2  
ping 10.112.97.2  
ping 10.112.33.2  
ping 10.112.164.2  
ping 10.112.224.130  

## 3. Router ke switch  
  
**R-DC**  
  
R-DC Fa0/1 ke SW-DC_CCTV-Kota Fa0/1  
R-DC Fa1/0 ke SW-DC_Server&Storage Fa0/1  
  
```
enable
configure terminal

interface FastEthernet0/1
 description LAN-CCTV-KOTA
 ip address 10.112.64.1 255.255.224.0
 no shutdown
exit

interface FastEthernet1/0
 description LAN-SERVER-STORAGE-DC
 ip address 10.112.96.1 255.255.255.0
 no shutdown
exit

end
write memory
```
  
Verifikasi  
  
Di R-DC:  
  
show ip interface brief  
ping 10.112.97.1  
ping 10.112.64.10  
ping 10.112.96.10  
  
Lalu dari R-PUSAT, uji jalur lewat static route:  
  
ping 10.112.64.1  
ping 10.112.96.1  
ping 10.112.96.10  
  
  
**R-BARAT**
  
R-Barat Fa0/1 ke SW-B_Wifi-Publik Fa0/1  
R-Barat Fa1/0 ke SW-B_Puskesmas&Sekolah Fa0/1  
  
```
enable
configure terminal

interface FastEthernet0/1
 description LAN-WIFI-PUBLIK-BARAT
 ip address 10.112.0.1 255.255.224.0
 no shutdown
exit

interface FastEthernet1/0
 description LAN-PUSKESMAS-SEKOLAH-BARAT
 ip address 10.112.32.1 255.255.255.0
 no shutdown
exit

end
write memory
```
  
Verifikasi  
  
Di R-BARAT:  
  
show ip interface brief  
ping 10.112.33.1  
ping 10.112.32.10  
ping 10.112.0.10  
  
Lalu dari R-PUSAT:  
  
ping 10.112.0.1  
ping 10.112.32.1  
ping 10.112.32.10  
  
**R-TIMUR**
  
R-Timur Fa0/1 ke SW-T_Wifi-Publik Fa0/1  
R-Timur Fa1/0 ke SW-T_Sensor-Banjir Fa0/1  

```
enable
configure terminal

interface FastEthernet0/1
 description LAN-WIFI-PUBLIK-TIMUR
 ip address 10.112.128.1 255.255.224.0
 no shutdown
exit

interface FastEthernet1/0
 description LAN-SENSOR-BANJIR-TIMUR
 ip address 10.112.160.1 255.255.252.0
 no shutdown
exit

end
write memory
```
  
Verifikasi  
  
Di R-TIMUR:  
  
show ip interface brief  
ping 10.112.164.1  
ping 10.112.128.10  
ping 10.112.160.10  
  
Lalu dari R-PUSAT:  
  
ping 10.112.128.1  
ping 10.112.160.1  
ping 10.112.160.10  
  
**R-SELATAN**
  
R-Selatan Fa0/1 ke SW-S_Wifi-Publik Fa0/1  
R-Selatan Fa1/0 ke SW-S_PJU-Pintar Fa0/1  
  
```
enable
configure terminal

interface FastEthernet0/1
 description LAN-WIFI-PUBLIK-SELATAN
 ip address 10.112.192.1 255.255.224.0
 no shutdown
exit

interface FastEthernet1/0
 description LAN-PJU-PINTAR-SELATAN
 ip address 10.112.224.1 255.255.255.128
 no shutdown
exit

end
write memory
```

Verifikasi  
  
Di R-SELATAN:  
  
show ip interface brief  
ping 10.112.224.129  
ping 10.112.192.10  
ping 10.112.224.10  
  
Lalu dari R-PUSAT:  
  
ping 10.112.192.1  
ping 10.112.224.1   
ping 10.112.224.10  
  
**R-UTARA**
  
R-UTARA0/1 ke R-PELABUHAN Fa0/0  
R-UTARA1/0 ke SW-U_Wifi-Publik Fa0/1  
R-UTARA1/1 ke SW-U_Sensor-Lalu-Lintas Fa0/1  
  
```
enable
configure terminal

interface FastEthernet0/1
 description L6-KE-R-PELABUHAN
 ip address 10.113.160.129 255.255.255.252
 no shutdown
exit

interface FastEthernet1/0
 description LAN-WIFI-PUBLIK-UTARA
 ip address 10.113.0.1 255.255.192.0
 no shutdown
exit

interface FastEthernet1/1
 description LAN-SENSOR-LALU-LINTAS-UTARA
 ip address 10.113.64.1 255.255.254.0
 no shutdown
exit

! Route ke seluruh blok Pelabuhan lewat L6
ip route 10.113.128.0 255.255.192.0 10.113.160.130

end
write memory
```
  
**R-PELABUHAN**
```
enable
configure terminal
hostname R-PELABUHAN
no ip domain-lookup

interface FastEthernet0/0
 description L6-KE-R-UTARA
 ip address 10.113.160.130 255.255.255.252
 no shutdown
exit

ip route 0.0.0.0 0.0.0.0 10.113.160.129

end
write memory
```
  
Verifikasi  
  
Di R-UTARA:  
  
show ip interface brief  
ping 10.113.66.1  
ping 10.113.160.130  
ping 10.113.0.10  
ping 10.113.64.10  
  
Lalu dari R-PUSAT:  
  
ping 10.113.0.1  
ping 10.113.64.1  
ping 10.113.160.130  
  
Ping ke 10.113.160.130 dari PUSAT melewati jalur Pelabuhan lewat L2, lalu R-UTARA, lalu L6. Kalau berhasil, route Pelabuhan sementara sudah benar (nanti diganti ke tunnel GRE). Terakhir, dari PC Sensor Lalu Lintas coba ping 8.8.8.8  
  
**R-PELABUHAN**
  
R-PELABUHAN Fa0/1 ke SW-Pel_Sensor&SCADA Fa0/1  
R-PELABUHAN Fa1/0 ke SW-Pel_Kantor-Pel Fa0/1  
  
```
enable
configure terminal

interface FastEthernet0/1
 description LAN-SENSOR-SCADA-PELABUHAN
 ip address 10.113.128.1 255.255.224.0
 no shutdown
exit

interface FastEthernet1/0
 description LAN-KANTOR-PELABUHAN
 ip address 10.113.160.1 255.255.255.128
 no shutdown
exit

end
write memory
```
  
Verifikasi  
  
Di R-PELABUHAN:  
  
show ip interface brief  
ping 10.113.160.129  
ping 10.113.128.10  
ping 10.113.160.10  
  
Lalu dari R-PUSAT:  
  
ping 10.113.128.1  
ping 10.113.160.1  

## 4. Menambahkan interface R-ISP ke default route

```
enable
configure terminal
hostname R-PUSAT
no ip domain-lookup

interface FastEthernet0/0
 description KE-R-ISP
 ip address 203.0.113.114 255.255.255.252
 ip nat outside
 no shutdown
exit

ip route 0.0.0.0 0.0.0.0 203.0.113.113

! NAT
ip access-list standard NAT-INTERNAL
 permit 10.112.0.0 0.1.255.255
exit

interface FastEthernet0/1
 ip nat inside
interface FastEthernet1/0
 ip nat inside
interface FastEthernet1/1
 ip nat inside
interface Ethernet0/0/0
 ip nat inside
interface Ethernet0/1/0
 ip nat inside
interface Ethernet0/2/0
 ip nat inside
interface Ethernet0/3/0
 ip nat inside
exit

ip nat inside source list NAT-INTERNAL interface FastEthernet0/0 overload
end
write memory
```
  

## 5. Pool DHCP di router

`excluded-address` mengecualikan gateway supaya tidak dibagikan ke host.

**R-PUSAT** (Pegawai Balai Kota)
```
enable
configure terminal
ip dhcp excluded-address 10.113.192.1
ip dhcp pool PEGAWAI-BALAIKOTA
 network 10.113.192.0 255.255.248.0
 default-router 10.113.192.1
 dns-server 10.112.96.2
exit
end
write memory
```

**R-BARAT**
```
enable
configure terminal
ip dhcp excluded-address 10.112.0.1
ip dhcp excluded-address 10.112.32.1
ip dhcp pool WIFI-BARAT
 network 10.112.0.0 255.255.224.0
 default-router 10.112.0.1
 dns-server 10.112.96.2
exit
ip dhcp pool PUSKESMAS-SEKOLAH
 network 10.112.32.0 255.255.255.0
 default-router 10.112.32.1
 dns-server 10.112.96.2
exit
end
write memory
```

**R-TIMUR**
```
enable
configure terminal
ip dhcp excluded-address 10.112.128.1
ip dhcp excluded-address 10.112.160.1
ip dhcp pool WIFI-TIMUR
 network 10.112.128.0 255.255.224.0
 default-router 10.112.128.1
 dns-server 10.112.96.2
exit
ip dhcp pool SENSOR-BANJIR
 network 10.112.160.0 255.255.252.0
 default-router 10.112.160.1
 dns-server 10.112.96.2
exit
end
write memory
```

**R-SELATAN**
```
enable
configure terminal
ip dhcp excluded-address 10.112.192.1
ip dhcp excluded-address 10.112.224.1
ip dhcp pool WIFI-SELATAN
 network 10.112.192.0 255.255.224.0
 default-router 10.112.192.1
 dns-server 10.112.96.2
exit
ip dhcp pool PJU-PINTAR
 network 10.112.224.0 255.255.255.128
 default-router 10.112.224.1
 dns-server 10.112.96.2
exit
end
write memory
```

**R-UTARA**
```
enable
configure terminal
ip dhcp excluded-address 10.113.0.1
ip dhcp excluded-address 10.113.64.1
ip dhcp pool WIFI-UTARA
 network 10.113.0.0 255.255.192.0
 default-router 10.113.0.1
 dns-server 10.112.96.2
exit
ip dhcp pool SENSOR-LALIN
 network 10.113.64.0 255.255.254.0
 default-router 10.113.64.1
 dns-server 10.112.96.2
exit
end
write memory
```

**R-PELABUHAN**
```
enable
configure terminal
ip dhcp excluded-address 10.113.128.1
ip dhcp excluded-address 10.113.160.1
ip dhcp pool SCADA-PELABUHAN
 network 10.113.128.0 255.255.224.0
 default-router 10.113.128.1
 dns-server 10.112.96.2
exit
ip dhcp pool KANTOR-PELABUHAN
 network 10.113.160.0 255.255.255.128
 default-router 10.113.160.1
 dns-server 10.112.96.2
exit
end
write memory
```

## 6. R-DC (relay untuk CCTV)

R-DC **tidak punya pool**, cukup ini:
```
enable
configure terminal
interface FastEthernet0/1
 ip helper-address 10.112.96.2
exit
end
write memory
```

## 7. Server DC (lewat GUI)

**IP statis**: klik server → Desktop → IP Configuration → Static
```
IP Address      : 10.112.96.2
Subnet Mask     : 255.255.255.0
Default Gateway : 10.112.96.1
DNS Server      : 10.112.96.2
```

**DHCP untuk CCTV**: klik server → Services → DHCP → **Service: On**, lalu isi:

| Field | Isi |
| :--- | :--- |
| Pool Name | CCTV-KOTA |
| Default Gateway | 10.112.64.1 |
| DNS Server | 10.112.96.2 |
| Start IP Address | 10.112.64.2 |
| Subnet Mask | 255.255.224.0 |
| Max Number of Users | 8000 |

Klik **Add**, lalu **Save**. Pool bawaan `serverPool` boleh dihapus (**Remove**) supaya tidak bentrok.

## 8. Server Balai Kota (statis)

Klik server → Desktop → IP Configuration → Static
```
IP Address      : 10.113.200.2
Subnet Mask     : 255.255.255.224
Default Gateway : 10.113.200.1
DNS Server      : 10.112.96.2
```

**Cara ngecek**

- Di PC/HP pengguna, pilih **DHCP** di IP Configuration. CCTV bakal dapat `10.112.64.2` dan seterusnya, dan DNS-nya `10.112.96.2`.
- Ping Server DC sekarang `10.112.96.2` (bukan `.10`).
- Dari tiap wilayah, ping `8.8.8.8`, lalu di R-PUSAT jalankan `show ip nat translations`.

Pastikan juga baris `ip route 10.112.0.0 255.254.0.0 ...` di R-ISP sudah dihapus. Itu cuma satu perintah:
```
R-ISP(config)# no ip route 10.112.0.0 255.254.0.0 203.0.113.114
``` 

Cek apakah Pool DHCP di masing masing router sudah dijalankan
```
show ip dhcp pool
```
Outputnya harus keluar nama pool 

* Set PC/HP di tiap LAN ke DHCP. Setiap PC harus dapat IP .2 dari subnetnya (contoh Pegawai dapat 10.113.192.2), gateway .1, dan DNS 10.112.96.2. 
* Lalu tes ping ke 10.112.96.2 dan 8.8.8.8.  ss

Tes translasi NAT, misal dari salah satu klien yaitu kantor pel  
1. Minta IP dhcp
2. Tunggu bentar
3. ping -t 8.8.8.8
4. Di router pusat ketik enable > show ip nat translations

## 9. Tunnel GRE

* Ujung tunnel memakai IP interface fisik L2 dan L6, yaitu `10.113.66.1` (PUSAT) dan `10.113.160.130` (PELABUHAN). Paket GRE tetap lewat jalur fisik PUSAT, UTARA, PELABUHAN.
* Route ke Pelabuhan lewat tunnel dibuat lebih spesifik dari route /18 lama, jadi router memilihnya lewat longest prefix match
* Route /18 via L2 sengaja tidak dihapus karena dibutuhkan agar ujung tunnel (10.113.160.130) tetap terjangkau lewat jalur fisik. Kalau route itu dihapus, tunnel bisa mati karena routing rekursif  
* R-PELABUHAN perlu route balik ke Balai Kota lewat tunnel. Tanpa itu, balasan lewat default route ke UTARA dan tidak melewati tunnel  

Di R-PUSAT
```
enable
configure terminal

interface Tunnel0
 description GRE-KE-R-PELABUHAN
 ip address 192.168.14.1 255.255.255.252
 tunnel source FastEthernet0/1
 tunnel destination 10.113.160.130
 ip nat inside
 no shutdown
exit

! Route Pelabuhan lewat tunnel (lebih spesifik dari /18 via L2)
ip route 10.113.128.0 255.255.224.0 192.168.14.2
ip route 10.113.160.0 255.255.255.128 192.168.14.2

end
write memory
```
  
Di R-PELABUHAN
```
enable
configure terminal

interface Tunnel0
 description GRE-KE-R-PUSAT
 ip address 192.168.14.2 255.255.255.252
 tunnel source FastEthernet0/0
 tunnel destination 10.113.66.1
 no shutdown
exit

! Route balik ke Balai Kota lewat tunnel
ip route 10.113.192.0 255.255.248.0 192.168.14.1
ip route 10.113.200.0 255.255.255.224 192.168.14.1

end
write memory
```
  
**Verifikasi**  
Tunggu sebentar sampai tunnel naik, lalu cek:
```
! R-PUSAT dan R-PELABUHAN
show ip interface brief       ! Tunnel0 harus up/up
show ip route                 ! ada route via 192.168.14.x

! R-PUSAT
ping 192.168.14.2
```
  
Lalu dari PC Pegawai Balai Kota, jalankan:
```
tracert 10.113.160.2
```
  
**Mengapa router memilih jalur tunnel dibandingkan jalur lain ke pelabuhan?**  
R-PUSAT punya dua route yang cocok untuk tujuan Pelabuhan, yaitu 10.113.128.0/18 via L2 dan /19 atau /25 via tunnel. Router selalu memilih prefix yang paling panjang (longest prefix match), jadi route via tunnel menang meski keduanya static dengan AD 1. R-PELABUHAN memilih tunnel ke Balai Kota karena /21 dan /27 lebih spesifik dari default route /0  
  
**Mengapa GRE saja tidak mengenkripsi data**  
GRE hanya encapsulation, membungkus paket asli dengan header GRE dan IP baru. Isi paket tetap plaintext tanpa enkripsi, autentikasi, atau pengecekan integritas, sehingga router di tengah (misalnya R-UTARA) tetap bisa membaca isinya. Karena itu ada bonus IPsec untuk melengkapinya  

**Next pasang IPsec**

Di R-PUSAT
```
enable
configure terminal

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit

crypto isakmp key K14SMARTCITY address 10.113.160.130

crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha-hmac
exit

! Trafik yang dienkripsi = paket GRE antar ujung tunnel
configure terminal
access-list 101 permit gre host 10.113.66.1 host 10.113.160.130

crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 10.113.160.130
 set transform-set TS-GRE
 match address 101
exit

interface FastEthernet0/1
 crypto map CMAP-GRE
exit

end
write memory
```

R-Pelabuhan : Sama, tapi alamatnya dibalik / mirror
```
enable
configure terminal

crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 2
 lifetime 86400
exit

crypto isakmp key K14SMARTCITY address 10.113.66.1

crypto ipsec transform-set TS-GRE esp-aes 256 esp-sha-hmac
exit

configure terminal
access-list 101 permit gre host 10.113.160.130 host 10.113.66.1

crypto map CMAP-GRE 10 ipsec-isakmp
 set peer 10.113.66.1
 set transform-set TS-GRE
 match address 101
exit

interface FastEthernet0/0
 crypto map CMAP-GRE
exit

end
write memory
```

Policy ISAKMP, key, dan transform-set harus sama persis di kedua sisi, dan ACL 101 harus mirror.
  
**Verifikasi**  
Pancing negosiasi dulu dengan ping dari R-PUSAT:
```
ping 192.168.14.2
```
Ping pertama biasanya .!!!! karena IPsec masih negosiasi, dan itu normal. Lalu cek:
```
show crypto isakmp sa
show crypto ipsec sa
```
show crypto isakmp sa harus menampilkan state QM_IDLE  
show crypto ipsec sa harus punya #pkts encaps dan #pkts decaps yang angkanya naik setiap melakukan ping  

## 10. Layanan Aplikasi Smart City

**DNS di LAN Server dan Storage DC. Nama www, ftp, dan mail dapat diakses dari seluruh wilayah**  
  
DNS dikerjakan paling awal karena web, FTP, dan email nantinya diakses menggunakan nama domain  
  
Rencana Penempatan Layanan  
  
| Layanan | Server | IP | Nama |
|---|---|---|---|
| DNS | Server-DC | 10.112.96.2 | Server nama |
| FTP | Server-DC | 10.112.96.2 | ftp.k14.smartcity.lab |
| Email | Server-DC | 10.112.96.2 | mail.k14.smartcity.lab |
| Web | Server Balai Kota | 10.113.200.2 | www.k14.smartcity.lab |
  
Satu server di Packet Tracer bisa menjalankan beberapa layanan sekaligus. Jadi, FTP dan email dapat dijalankan di Server-DC   
Soal hanya mewajibkan DNS dan FTP berada di LAN Server & Storage DC, sedangkan lokasi email bebas  
  
Langkah 1: Pastikan Prasyarat    
  
- **Server-DC** sudah menggunakan IP statis:
  - IP: `10.112.96.2`
  - Subnet Mask: `255.255.255.0`
  - Gateway: `10.112.96.1`
  - DNS: `10.112.96.2`
  
- **Server Balai Kota** sudah menggunakan IP statis:
  - IP: `10.113.200.2`
  - Subnet Mask: `255.255.255.224`
  - Gateway: `10.113.200.1`
  - DNS: `10.112.96.2`
  
- **PC pengguna** sudah mendapatkan IP melalui DHCP dengan DNS `10.112.96.2`  
  
Jika PC mendapatkan IP sebelum DNS diatur, lakukan ulang DHCP dengan memilih **Static**, kemudian **DHCP** kembali   
  
Langkah 2: Aktifkan DNS di Server-DC  
1. Klik **Server-DC**  
2. Buka tab **Services**  
3. Pilih **DNS**  
4. Pastikan **DNS Service** berada pada posisi **On**  
  
Langkah 3: Tambahkan 3 Record DNS  
  
Isi formulir DNS sesuai tabel berikut, kemudian klik **Add** untuk setiap record  
  
| Name | Type | Address |
|---|---|---|
| `www.k14.smartcity.lab` | A Record | `10.113.200.2` |
| `ftp.k14.smartcity.lab` | A Record | `10.112.96.2` |
| `mail.k14.smartcity.lab` | A Record | `10.112.96.2` |
  
A Record adalah jenis record DNS yang memetakan nama ke alamat IPv4    
  
Langkah 4: Tes DNS dari PC   
  
Buka **PC → Desktop → Command Prompt**, kemudian jalankan:  
  
```bash
nslookup www.k14.smartcity.lab
nslookup ftp.k14.smartcity.lab
nslookup mail.k14.smartcity.lab
```
  
Contoh hasil yang diharapkan:  
  
```text
Server: 10.112.96.2
Address: 10.112.96.2

Name: www.k14.smartcity.lab
Address: 10.113.200.2
```
  
**Web server balai kota** 
  
DNS sudah siap dengan konfigurasi `www.k14.smartcity.lab → 10.113.200.2`. Sekarang kita tinggal mengaktifkan layanan web dan membuat halaman utamanya  
  
Langkah 1: Aktifkan HTTP  
1. Klik **Server Balai Kota**.
2. Buka tab **Services → HTTP**.
3. Pastikan layanan **HTTP** berada pada posisi **On**.
4. **HTTPS** boleh dibiarkan karena tidak wajib untuk tugas ini.
  
Langkah 2: Edit Halaman index.html  
  
Pada tabel file di menu HTTP, terdapat file bawaan bernama `index.html`  
1. Klik **Edit** di sebelah file `index.html`, atau klik nama file lalu pilih **Edit**.
2. Hapus seluruh isi file tersebut.
3. Ganti dengan kode HTML berikut:
  
```html

<html>
<head>
<title>Smart City Surabaya | Hacker Terminal</title>
</head>

<body bgcolor="#050B08" text="#00FF66">

<center>
<br>

<table width="85%" border="1" bordercolor="#00FF66" bgcolor="#09150D" cellpadding="15">

<tr>
<td>

<font face="Courier New" color="#00FF66" size="3">
<b>
[ SYSTEM TERMINAL ]<br>
[ SMART CITY SURABAYA ]
</b>
</font>

<hr color="#00FF66">

<center>

<font face="Courier New" color="#00FF66" size="6">
<b>PORTAL SMART CITY SURABAYA</b>
</font>

<br><br>

<font face="Courier New" color="#66FF99" size="5">
<b>[ KELOMPOK 14 ]</b>
</font>

<br><br>

<font face="Courier New" color="#00FF66" size="3">
================================<br>
WELCOME TO SMART CITY NETWORK<br>
================================
</font>

<br><br>

<font face="Courier New" color="#FFFFFF" size="3">
Selamat datang di portal resmi<br>
jaringan Smart City Kota Surabaya.
</font>

<br><br>

</center>

<hr color="#00FF66">

<font face="Courier New" color="#00FF66" size="3">

<b>root@smartcity:~$ system_info</b>

<br><br>

[+] DOMAIN : k14.smartcity.lab
<br><br>

[+] SERVER : Server Balai Kota
<br><br>

[+] IP ADDRESS : 10.113.200.2
<br><br>

[+] NETWORK : SMART CITY SURABAYA
<br><br>

[+] SERVICES :
<br>
&nbsp;&nbsp;&nbsp;> WEB
<br>
&nbsp;&nbsp;&nbsp;> DNS
<br>
&nbsp;&nbsp;&nbsp;> FTP
<br>
&nbsp;&nbsp;&nbsp;> EMAIL

</font>

<br><br>

<hr color="#00FF66">

<center>

<font face="Courier New" color="#66FF99" size="3">
<b>
[ SMART CITY NETWORK PORTAL ]
</b>
</font>

<br><br>

<font face="Courier New" color="#00FF66" size="2">
--- KELOMPOK 14 ---<br>
Surabaya Smart City Infrastructure
</font>

</center>

</td>
</tr>
</table>

<br>

<font face="Courier New" color="#00FF66" size="2">
root@smartcity:~$ _
</font>

</center>

</body>
</html>

```

4. Klik **Save** untuk menyimpan perubahan  
- Tombol **Save** wajib diklik agar perubahan tersimpan  
- Packet Tracer mendukung HTML sederhana, jadi tidak perlu menggunakan CSS atau JavaScript yang rumit  
- Pastikan tulisan **Portal Smart City Surabaya** dan **Kelompok 14** muncul di halaman karena keduanya menjadi bagian penilaian  
  

Langkah 3: Tes Web dari PC  
1. Klik salah satu PC  
2. Buka **Desktop → Web Browser**  
3. Masukkan alamat berikut:

   `www.k14.smartcity.lab`
4. Klik **Go**  
5. Pastikan halaman **Portal Smart City Surabaya** muncul  
  
Selanjutnya, lakukan pengujian menggunakan alamat IP langsung:  
  
`http://10.113.200.2`  
  
Pengujian ini membantu kita mengetahui apakah masalah berasal dari web atau DNS    

| Hasil Pengujian | Artinya |
|---|---|
| Nama domain dan IP berhasil | Web dan DNS sudah berfungsi. |
| IP berhasil, tetapi nama domain gagal | Ada masalah DNS. Periksa menggunakan `nslookup`. |
| Keduanya gagal | HTTP belum aktif atau terdapat masalah koneksi ke server. |
  
Langkah 4: Tes Web dari Semua Wilayah  
  
Buka `www.k14.smartcity.lab` melalui Web Browser dari minimal satu PC di setiap wilayah  

| Wilayah | PC yang Diuji |
|---|---|
| Balai Kota | PC Pegawai |
| DC | PC CCTV |
| Barat | PC WiFi atau Puskesmas |
| Timur | PC WiFi atau Sensor Banjir |
| Selatan | PC WiFi atau PJU |
| Utara | PC WiFi atau Sensor Lalu Lintas |
| Pelabuhan | PC Kantor atau SCADA |

**Catatan:**

Wilayah yang konfigurasi pool DHCP-nya belum dibuat (Pusat, Timur, Selatan, Utara, dan Pelabuhan) belum bisa mendapatkan alamat DNS secara otomatis  
  
Selesaikan konfigurasi DHCP terlebih dahulu atau isi DNS secara manual untuk sementara  
  
Untuk perangkat HP atau smartphone yang terhubung ke LAN WiFi, buka aplikasi **Web Browser** yang tersedia pada perangkat tersebut dan gunakan URL yang sama  
  
Troubleshooting jika Web Tidak Berfungsi
| Gejala | Penyebab / Solusi |
|---|---|
| Halaman bawaan Cisco Packet Tracer muncul | File `index.html` belum diedit atau perubahan belum disimpan menggunakan Save. |
| `Request Timeout` | HTTP belum aktif atau terdapat masalah routing ke `10.113.200.2`. |
| Web hanya bisa diakses melalui IP | Periksa konfigurasi DNS pada PC dan record `www.k14.smartcity.lab`. |
| Web gagal di satu wilayah saja | Kemungkinan terdapat masalah DHCP atau routing di wilayah tersebut. |

Untuk memeriksa koneksi menuju server, buka **Command Prompt** pada PC dan jalankan:
```bash
ping 10.113.200.2
```
Jika ping berhasil tetapi halaman web tidak muncul, periksa kembali layanan HTTP dan file `index.html`  
Jika web bisa diakses melalui IP tetapi gagal melalui nama domain, jalankan:
```bash
nslookup www.k14.smartcity.lab
```
Pastikan hasilnya menunjukkan alamat IP `10.113.200.2`

Ambil screenshot berikut sebagai bukti konfigurasi:

1. **Services → HTTP** pada Server Balai Kota yang memperlihatkan HTTP dalam posisi On.
2. Isi file `index.html` yang sudah diedit dan disimpan.
3. Tampilan Web Browser yang menunjukkan halaman **Portal Smart City Surabaya**.
4. Hasil akses web dari minimal tiga wilayah berbeda, termasuk Balai Kota dan Pelabuhan, dengan URL domain terlihat.

**FTP di LAN Server dan Storage DC sebagai penyimpanan rekaman CCTV, dengan akun login. Perangkat di LAN CCTV Kota dapat mengunggah file ke sana**  
  
FTP jalan di Server-DC (10.112.96.2), nama sudah sesuai A record ftp.k14.smartcity.lab yang telah dibuat sebelumnya. Sekarang kita tes dengan melakukan login & unggah file dari PC CCTV  

Langkah 1: Aktifkan FTP dan Buat Akun

1. Klik **Server-DC**  
2. Buka tab **Services → FTP**  
3. Pastikan **FTP Service** berada pada posisi **On**   
4. Pada bagian **User Setup**, isi konfigurasi berikut:  
| Field | Isi |
|---|---|
| Username | `cctv` |
| Password | `cctv14` |
| Permission | **Write**, **Read**, dan **List** |
5. Centang permission **Write**, **Read**, dan **List**  
6. Klik **Add** untuk menambahkan akun  
7. Pastikan akun `cctv` muncul pada tabel pengguna FTP  

Langkah 2: Siapkan File Rekaman di PC CCTV  
  
PC CCTV membutuhkan file untuk diunggah ke FTP Server   
  
Karena rekaman video tidak disimulasikan secara langsung, kita menggunakan file teks sebagai representasi rekaman CCTV 
1. Klik **PC CCTV**.
2. Buka **Desktop → Text Editor**.
3. Masukkan teks berikut:
```text
Rekaman CCTV Kota Surabaya
Kamera: 01
Tanggal: 02-11-2026
Lokasi: Smart City Surabaya
Status: Rekaman berhasil disimpan
```
4. Klik **File → Save**  
5. Simpan file menggunakan nama:

   `rekaman_cctv01.txt`
  
Pastikan file sudah tersimpan sebelum melakukan proses unggah  
  
Langkah 3: Unggah File Melalui FTP6. Login menggunakan akun FTP yang sudah dibuat

Lakukan pengujian menggunakan perintah `put` dan `dir`  
Pengujian berhasil jika PC dapat login dan mengunggah file ke FTP Server  
  
Troubleshooting: Jika FTP Tidak Berfungsi  
| Gejala | Penyebab / Solusi |
|---|---|
| `Request timed out` atau tidak terhubung | FTP Service belum aktif atau koneksi menuju server bermasalah. |
| Bisa ping tetapi nama domain tidak dikenali | Periksa DNS menggunakan `nslookup` atau coba FTP melalui IP. |
| `Invalid username or password` | Akun belum ditambahkan atau username dan password salah. |
| Login berhasil tetapi `put` ditolak | Permission **Write** belum diaktifkan. |
| File tidak ditemukan saat menggunakan `put` | File belum disimpan atau nama file tidak sesuai. |
| PC CCTV tidak mendapatkan DNS | Ulangi DHCP dan pastikan DNS mengarah ke `10.112.96.2`. |

Untuk memastikan PC CCTV dapat menjangkau Server-DC, jalankan:

```bash
ping 10.112.96.2
```
Jika ping berhasil tetapi FTP tidak terhubung, periksa kembali layanan FTP dan akun pengguna  
  
Untuk memeriksa nama domain FTP, jalankan:
```bash
nslookup ftp.k14.smartcity.lab
```
Pastikan hasilnya menunjukkan alamat IP:
```text
10.112.96.2
```
  
Jika nama domain gagal tetapi koneksi melalui IP berhasil, kemungkinan masalah terdapat pada konfigurasi DNS  
  
Ambil screenshot berikut sebagai bukti konfigurasi:  
1. **Services → FTP** pada Server-DC yang memperlihatkan FTP Service dalam posisi On dan akun `cctv` pada tabel pengguna.
2. Isi file `rekaman_cctv01.txt` pada Text Editor PC CCTV.
3. Tampilan Command Prompt PC CCTV yang menunjukkan proses login FTP.
4. Hasil perintah `put` yang menunjukkan file berhasil diunggah.
5. Hasil perintah `dir` yang menampilkan file `rekaman_cctv01.txt`.
6. Opsional: hasil pengujian FTP dari PC di wilayah lain.

**Email antarpegawai. Pengguna di Balai Kota dan Pelabuhan dapat saling mengirim dan menerima email**

Server email dijalankan di **Server-DC** dengan IP `10.112.96.2` dan menggunakan domain `mail.k14.smartcity.lab` yang sudah dikonfigurasi pada DNS  
  
Kita akan membuat akun email untuk pegawai Balai Kota dan Pelabuhan, mengatur aplikasi Email pada masing-masing PC, lalu menguji pengiriman dan penerimaan email dua arah  
  
Langkah 1: Aktifkan Layanan Email di Server-DC  
1. Klik **Server-DC**.
2. Buka tab **Services → EMAIL**.
3. Pastikan kedua layanan berikut berada pada posisi **On**:
   - **SMTP (Simple Mail Transfer Protocol):** Digunakan untuk mengirim email.
   - **POP3 (Post Office Protocol version 3):** Digunakan untuk mengambil email dari server.
4. Pada kolom **Domain Name**, masukkan:

   ```text
   k14.smartcity.lab
   ```

5. Klik **Set** untuk menyimpan domain.

**Catatan:** Tombol **Set** wajib diklik agar domain tersimpan. Jika tidak, alamat email yang menggunakan domain tersebut tidak akan dikenali oleh server  
  
Langkah 2: Buat Akun Pengguna Email  
  
Masih pada halaman **Services → EMAIL**, cari bagian **User Setup**  
Tambahkan dua akun berikut:  
| User | Password | Digunakan di |
|---|---|---|
| `pegawai` | `pegawai14` | PC Pegawai Balai Kota |
| `pelabuhan` | `pelabuhan14` | PC Kantor Pelabuhan |
  
Langkah pembuatan akun:  
1. Masukkan **User** sesuai tabel.
2. Masukkan **Password**.
3. Klik tombol **+** untuk menambahkan akun.
4. Pastikan akun muncul pada tabel pengguna.
5. Ulangi langkah tersebut untuk akun berikutnya.
  
Setelah berhasil dibuat, alamat email lengkapnya menjadi:  
- **Balai Kota:** `pegawai@k14.smartcity.lab`
- **Pelabuhan:** `pelabuhan@k14.smartcity.lab`
  
**Catatan:** Dua akun tersebut sudah cukup untuk memenuhi kebutuhan pengujian. Jika diperlukan, kita juga dapat menambahkan akun lain seperti `pegawai2` atau `pelabuhan2`  
  
Langkah 3: Konfigurasi Email di PC Pegawai Balai Kota  
  
1. Klik **PC Pegawai Balai Kota**.
2. Buka **Desktop → Email**.
3. Pada halaman **Configure Mail**, isi konfigurasi berikut:  
| Field | Isi |
|---|---|
| Your Name | Pegawai Balai Kota |
| Email Address | `pegawai@k14.smartcity.lab` |
| Incoming Mail Server | `mail.k14.smartcity.lab` |
| Outgoing Mail Server | `mail.k14.smartcity.lab` |
| User Name | `pegawai` |
| Password | `pegawai14` |

4. Klik **Save** untuk menyimpan konfigurasi  
  
**Penjelasan:**  
- **Your Name:** Nama pengirim yang ditampilkan pada email
- **Email Address:** Alamat email lengkap pengguna
- **Incoming Mail Server:** Server untuk menerima email melalui POP3
- **Outgoing Mail Server:** Server untuk mengirim email melalui SMTP
- **User Name:** Nama akun yang terdaftar pada server
- **Password:** Kata sandi akun email
  
Gunakan nama domain `mail.k14.smartcity.lab` agar konfigurasi email sekaligus memanfaatkan layanan DNS yang sudah dibuat  
  
Langkah 4: Konfigurasi Email di PC Kantor Pelabuhan 
  
Lakukan konfigurasi yang sama pada **PC Kantor Pelabuhan **  
  
1. Klik **PC Kantor Pelabuhan**
2. Buka **Desktop → Email**
3. Isi konfigurasi berikut:  
| Field | Isi |
|---|---|
| Your Name | Pegawai Pelabuhan |
| Email Address | `pelabuhan@k14.smartcity.lab` |
| Incoming Mail Server | `mail.k14.smartcity.lab` |
| Outgoing Mail Server | `mail.k14.smartcity.lab` |
| User Name | `pelabuhan` |
| Password | `pelabuhan14` |
4. Klik **Save**  
**Catatan:**  
Pastikan PC Pelabuhan sudah mendapatkan konfigurasi jaringan melalui DHCP, termasuk:
- Alamat IP sesuai subnet Pelabuhan.
- Default Gateway sesuai router Pelabuhan.
- DNS Server `10.112.96.2`.
  
Jika PC belum mendapatkan DNS, periksa konfigurasi pool DHCP pada **R-PELABUHAN** dan lakukan pembaruan DHCP.  
  
Langkah 5: Tes Pengiriman dan Penerimaan Email Dua Arah  
  
Pengujian dilakukan dalam dua arah untuk membuktikan bahwa pegawai Balai Kota dan Pelabuhan dapat saling mengirim serta menerima email  

5.A. Pengiriman dari Balai Kota ke Pelabuhan  
  
1. Buka **PC Pegawai Balai Kota → Desktop → Email**
2. Klik **Compose** untuk membuat email baru
3. Isi email sebagai berikut:

   | Field | Isi |
   |---|---|
   | To | `pelabuhan@k14.smartcity.lab` |
   | Subject | `Tes dari Balai Kota` |
   | Message | `Halo Pelabuhan, ini email uji dari Balai Kota.` |

4. Klik **Send**
5. Pastikan muncul pesan bahwa email berhasil dikirim
6. Buka **PC Kantor Pelabuhan → Desktop → Email**
7. Klik **Receive** untuk mengambil email dari server
8. Pastikan email dari Balai Kota muncul pada Inbox
9. Klik email tersebut untuk membaca isinya

**Hasil yang diharapkan:** Email dari `pegawai@k14.smartcity.lab` berhasil diterima oleh `pelabuhan@k14.smartcity.lab`.

5.B. Pengiriman dari Pelabuhan ke Balai Kota

1. Buka **PC Kantor Pelabuhan → Desktop → Email**
2. Klik **Compose**
3. Isi email sebagai berikut:

   | Field | Isi |
   |---|---|
   | To | `pegawai@k14.smartcity.lab` |
   | Subject | `Balasan dari Pelabuhan` |
   | Message | `Halo Balai Kota, email sudah diterima. Ini balasan dari Pelabuhan.` |

4. Klik **Send**
5. Pastikan email berhasil dikirim
6. Buka **PC Pegawai Balai Kota → Desktop → Email**
7. Klik **Receive**
8. Pastikan email balasan dari Pelabuhan muncul pada Inbox
  
**Hasil yang diharapkan:** Email dari `pelabuhan@k14.smartcity.lab` berhasil diterima oleh `pegawai@k14.smartcity.lab`.
  
**Catatan Penting:**  
- Pengujian wajib dilakukan dua arah karena soal meminta kedua wilayah dapat saling mengirim dan menerima email.
- Email yang masuk biasanya baru ditampilkan setelah pengguna menekan tombol **Receive**.
- Jika kedua pengujian berhasil, layanan SMTP dan POP3 sudah berfungsi sesuai kebutuhan.

Troubleshooting: Jika Email Tidak Berfungsi

| Gejala | Penyebab / Solusi |
|---|---|
| Error saat Send atau Receive | SMTP atau POP3 belum aktif, atau domain belum disimpan menggunakan tombol Set. |
| `Unable to connect to server` | Nama domain tidak dikenali DNS atau koneksi menuju server bermasalah. |
| `Authentication failed` | User Name atau Password pada PC tidak sesuai dengan akun di Server-DC. |
| Email berhasil dikirim tetapi tidak diterima | Tombol Receive belum diklik atau alamat penerima salah. |
| Alamat penerima ditolak | Akun penerima belum ditambahkan ke server menggunakan tombol +. |
| PC Pelabuhan tidak mendapatkan DNS | Pool DHCP pada R-PELABUHAN belum dikonfigurasi atau DHCP perlu diperbarui. |

Jika aplikasi Email tidak dapat terhubung ke server, buka **Desktop → Command Prompt** dan jalankan:

```bash
ping 10.112.96.2
```
  
Jika ping berhasil, periksa DNS menggunakan:

```bash
nslookup mail.k14.smartcity.lab
```
  
Pastikan nama domain mengarah ke alamat:

```text
10.112.96.2
```

Jika DNS gagal tetapi server dapat dijangkau melalui IP, coba gunakan `10.112.96.2` sebagai Incoming dan Outgoing Mail Server untuk sementara  
Jika koneksi melalui IP berhasil, masalah kemungkinan terdapat pada konfigurasi DNS  
  
Pastikan informasi akun sesuai dengan yang dibuat pada Server-DC:
- Username hanya menggunakan `pegawai` atau `pelabuhan`.
- Jangan memasukkan alamat email lengkap pada kolom User Name.
- Password harus sama persis dengan yang terdaftar pada server.
  
Ambil screenshot berikut sebagai bukti konfigurasi dan pengujian:

1. **Services → EMAIL** pada Server-DC yang menunjukkan SMTP dan POP3 dalam posisi On.
2. Domain `k14.smartcity.lab` dan kedua akun pengguna pada tabel User Setup.
3. Konfigurasi Email pada PC Pegawai Balai Kota.
4. Konfigurasi Email pada PC Kantor Pelabuhan.
5. Email yang berhasil dikirim dari Balai Kota ke Pelabuhan.
6. Inbox Pelabuhan yang menampilkan email dari Balai Kota.
7. Email balasan yang berhasil dikirim dari Pelabuhan ke Balai Kota.
8. Inbox Balai Kota yang menampilkan email balasan dari Pelabuhan.
  
Setelah seluruh pengujian berhasil, empat layanan utama telah dikonfigurasi:
| Layanan | Lokasi Server | Domain |
|---|---|---|
| DNS | Server-DC | `k14.smartcity.lab` |
| Web | Server Balai Kota | `www.k14.smartcity.lab` |
| FTP | Server-DC | `ftp.k14.smartcity.lab` |
| Email | Server-DC | `mail.k14.smartcity.lab` |

Konfigurasi **Poin 8** dinyatakan selesai apabila:

- DNS dapat menerjemahkan nama domain dari seluruh wilayah.
- Web dapat diakses melalui nama domain dari seluruh wilayah.
- FTP dapat digunakan untuk mengunggah file dari PC CCTV.
- Email dapat dikirim dan diterima dua arah antara Balai Kota dan Pelabuhan.

## 11. OSPF

**Langkah 1: Menambahkan Link L7 (R-DC ke R-UTARA)**

Link L7 digunakan sebagai jalur cadangan antara **R-DC** dan **R-UTARA**  
Alamat jaringan yang digunakan adalah `10.112.97.4/30`  
  
Blok ini masih tersedia dalam alokasi DC1 `/23`, karena jaringan tersebut sudah menggunakan:
- Server DC: `10.112.96.0/24`
- Link L1: `10.112.97.0/30`
- Link L7: `10.112.97.4/30`
  
**Konfigurasi IP Link L7**  
| Router | Interface | IP Address |
|---|---|---|
| R-DC | Ethernet0/1/0 | `10.112.97.5/30` |
| R-UTARA | Ethernet0/0/0 | `10.112.97.6/30` |
  
Subnet mask yang digunakan adalah `255.255.255.252`  
  
**A. Menambahkan Modul Ethernet**  
  
1. Klik router yang akan dikonfigurasi
2. Buka tab **Physical**
3. Matikan router menggunakan tombol power
4. Seret modul **WIC-1ENET** ke slot WIC yang masih kosong
5. Nyalakan kembali router
6. Periksa nama interface menggunakan perintah:

```bash
enable
show ip interface brief
```
  
Penempatan modul:
- **R-DC:** Modul baru dipasang pada slot 1 sehingga menghasilkan interface `Ethernet0/1/0`
- **R-UTARA:** Modul dipasang pada slot 0 sehingga menghasilkan interface `Ethernet0/0/0`
  
Nama interface dapat berbeda tergantung posisi slot. Sesuaikan konfigurasi dengan hasil `show ip interface brief`  
  
**B. Menghubungkan Kedua Router**  
  
Gunakan kabel **Copper Cross-Over** untuk menghubungkan `R-DC Ethernet0/1/0 → R-UTARA Ethernet0/0/0`  
  
**C. Konfigurasi R-DC**  
  
Masukkan perintah berikut pada CLI R-DC:  
```bash
enable
configure terminal
interface Ethernet0/1/0
 description L7-KE-R-UTARA
 ip address 10.112.97.5 255.255.255.252
 no shutdown
end
write memory
```
  
**D. Konfigurasi R-UTARA**
  
Masukkan perintah berikut pada CLI R-UTARA:
```bash
enable
configure terminal
interface Ethernet0/0/0
 description L7-KE-R-DC
 ip address 10.112.97.6 255.255.255.252
 no shutdown
end
write memory
```
  
**E. Pengujian Link L7**  
  
Dari R-DC, jalankan:
```bash
ping 10.112.97.6
```
  
Pengujian berhasil jika R-DC mendapatkan balasan dari R-UTARA  
  
Jika ping gagal, periksa:
- Kondisi kabel antar-router.
- Status interface menggunakan `show ip interface brief`.
- Kesesuaian IP address dan subnet mask.
- Pastikan kedua interface sudah menggunakan `no shutdown`.
  
**Langkah 2: Konfigurasi Routing OSPF**  
  
OSPF (Open Shortest Path First) adalah protokol routing dinamis yang digunakan agar router dapat bertukar informasi jaringan dan menentukan jalur terbaik secara otomatis  
  
Seluruh router menggunakan **OSPF Area 0**  
  
**Ketentuan Konfigurasi:**
- Semua router menggunakan OSPF process ID `1`.
- Setiap router memiliki Router ID yang berbeda.
- Interface LAN menggunakan **passive-interface** agar jaringan tetap diumumkan tanpa mengirim paket Hello OSPF ke perangkat pengguna.
- Interface antar-router menggunakan **no passive-interface** agar dapat membentuk hubungan OSPF.
- Interface menuju ISP pada R-PUSAT tidak dimasukkan ke OSPF.
- R-ISP tidak mengikuti OSPF dan tidak perlu diubah.
  
**A. Konfigurasi R-PUSAT**  
  
Router ID: `1.1.1.1`
  
```bash
enable
configure terminal
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface FastEthernet0/1
 no passive-interface Ethernet0/0/0
 no passive-interface Ethernet0/1/0
 no passive-interface Ethernet0/2/0
 no passive-interface Ethernet0/3/0
 no passive-interface Tunnel0
 network 10.113.66.0 0.0.0.3 area 0
 network 10.112.97.0 0.0.0.3 area 0
 network 10.112.33.0 0.0.0.3 area 0
 network 10.112.164.0 0.0.0.3 area 0
 network 10.112.224.128 0.0.0.3 area 0
 network 192.168.14.0 0.0.0.3 area 0
 network 10.113.192.0 0.0.7.255 area 0
 network 10.113.200.0 0.0.0.31 area 0
 default-information originate
exit
end
write memory
```
  
Perintah `default-information originate` digunakan untuk mengumumkan default route dari R-PUSAT ke router OSPF lainnya, selama default route tersedia pada tabel routing R-PUSAT  
  
**B. Konfigurasi R-DC**  
  
Router ID: `2.2.2.2`
  
```bash
enable
configure terminal
router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface Ethernet0/0/0
 no passive-interface Ethernet0/1/0
 network 10.112.97.0 0.0.0.3 area 0
 network 10.112.97.4 0.0.0.3 area 0
 network 10.112.64.0 0.0.31.255 area 0
 network 10.112.96.0 0.0.0.255 area 0
exit
end
write memory
```
  
**C. Konfigurasi R-BARAT**  
  
Router ID: `3.3.3.3`  
  
```bash
enable
configure terminal
router ospf 1
 router-id 3.3.3.3
 passive-interface default
 no passive-interface Ethernet0/0/0
 network 10.112.33.0 0.0.0.3 area 0
 network 10.112.0.0 0.0.31.255 area 0
 network 10.112.32.0 0.0.0.255 area 0
exit
end
write memory
```
  
**D. Konfigurasi R-TIMUR**  
  
Router ID: `4.4.4.4`  
  
```bash
enable
configure terminal
router ospf 1
 router-id 4.4.4.4
 passive-interface default
 no passive-interface Ethernet0/0/0
 network 10.112.164.0 0.0.0.3 area 0
 network 10.112.128.0 0.0.31.255 area 0
 network 10.112.160.0 0.0.3.255 area 0
exit
end
write memory
```
  
**E. Konfigurasi R-SELATAN**  
  
Router ID: `5.5.5.5`  
  
```bash
enable
configure terminal
router ospf 1
 router-id 5.5.5.5
 passive-interface default
 no passive-interface Ethernet0/0/0
 network 10.112.224.128 0.0.0.3 area 0
 network 10.112.192.0 0.0.31.255 area 0
 network 10.112.224.0 0.0.0.127 area 0
exit
end
write memory
```
  
**F. Konfigurasi R-UTARA**  
  
Router ID: `6.6.6.6`  
  
```bash
enable
configure terminal
router ospf 1
 router-id 6.6.6.6
 passive-interface default
 no passive-interface FastEthernet0/0
 no passive-interface FastEthernet0/1
 no passive-interface Ethernet0/0/0
 network 10.113.66.0 0.0.0.3 area 0
 network 10.113.160.128 0.0.0.3 area 0
 network 10.112.97.4 0.0.0.3 area 0
 network 10.113.0.0 0.0.63.255 area 0
 network 10.113.64.0 0.0.1.255 area 0
exit
end
write memory
```
  
**G. Konfigurasi R-PELABUHAN**  
  
Router ID: `7.7.7.7`  
  
```bash
enable
configure terminal
router ospf 1
 router-id 7.7.7.7
 passive-interface default
 no passive-interface FastEthernet0/0
 no passive-interface Tunnel0
 network 10.113.160.128 0.0.0.3 area 0
 network 192.168.14.0 0.0.0.3 area 0
 network 10.113.128.0 0.0.31.255 area 0
 network 10.113.160.0 0.0.0.127 area 0
exit
end
write memory
```
  
**H. Konfigurasi R-ISP**  
  
R-ISP tidak perlu dikonfigurasi ulang  
  
Router tersebut:
- Tidak mengikuti OSPF.
- Tidak menerima informasi jaringan privat melalui OSPF.
- Tetap menggunakan konfigurasi sebelumnya.

**Langkah 3: Mengatur OSPF Cost untuk Jalur Tunnel**  
  
OSPF menggunakan **cost** sebagai nilai untuk menentukan jalur terbaik. Semakin kecil total cost, semakin diprioritaskan jalur tersebut  
  
Pada topologi ini, trafik antara Balai Kota dan Pelabuhan diharapkan tetap menggunakan Tunnel0 selama jalur tersedia  
  
Namun, penggunaan OSPF pada tunnel dapat menimbulkan **recursive routing**, yaitu kondisi ketika router mencoba mencapai alamat tujuan tunnel melalui tunnel itu sendiri  
  
Jika terjadi, tunnel dapat mati dengan pesan:

```text
Tunnel0 temporarily disabled due to recursive routing
```
  
Untuk menghindarinya, gunakan ketentuan:  
`Cost L2 dan L6 < Cost Tunnel < Cost L2 + Cost L6`
  
Dengan konfigurasi:
| Link | OSPF Cost |
|---|---|
| L2 (R-PUSAT ke R-UTARA) | 10 |
| L6 (R-UTARA ke R-PELABUHAN) | 10 |
| Tunnel0 (R-PUSAT ke R-PELABUHAN) | 15 |
  
**A. Konfigurasi Cost R-PUSAT**  
```bash
enable
configure terminal
interface FastEthernet0/1
 ip ospf cost 10
interface Tunnel0
 ip ospf cost 15
end
write memory
```

**B. Konfigurasi Cost R-UTARA**
```bash
enable
configure terminal
interface FastEthernet0/0
 ip ospf cost 10
interface FastEthernet0/1
 ip ospf cost 10
end
write memory
```

**C. Konfigurasi Cost R-PELABUHAN**
```bash
enable
configure terminal
interface FastEthernet0/0
 ip ospf cost 10
interface Tunnel0
 ip ospf cost 15
end
write memory
```

**Penjelasan Pemilihan Jalur**
Dengan cost tersebut, router dapat membedakan jalur fisik untuk mencapai endpoint tunnel dan jalur tunnel untuk mencapai jaringan Pelabuhan  
  
Dalam topologi ini, kombinasi cost dirancang agar:  
- Endpoint tunnel tetap dicapai melalui jalur fisik.
- Trafik dari Balai Kota ke LAN Pelabuhan lebih diprioritaskan melalui Tunnel0.
- Link L7 menjadi jalur cadangan ketika jalur utama mengalami gangguan.
  
Link Ethernet L1, L3, L4, L5, dan L7 menggunakan cost bawaan 10 apabila interface berkecepatan 10 Mbps dan reference bandwidth OSPF masih menggunakan nilai standar 100 Mbps  
  
**Langkah 4: Menghapus Static Route Antar-Router**  
  
Setelah konfigurasi OSPF selesai, static route antar-router dihapus agar proses pemilihan jalur dilakukan menggunakan routing dinamis  
  
**Penting:** Pastikan OSPF sudah membentuk neighbor dan mempelajari rute yang diperlukan sebelum menghapus static route  
  
Untuk melihat static route yang sedang digunakan, jalankan pada masing-masing router:
```bash
show running-config
```
  
Atau gunakan perintah berikut jika didukung oleh IOS:
```bash
show run | include ip route
```
  
**A. Menghapus Static Route R-PUSAT**
  
```bash
enable
configure terminal
no ip route 10.112.0.0 255.255.192.0 10.112.33.2
no ip route 10.112.64.0 255.255.192.0 10.112.97.2
no ip route 10.112.128.0 255.255.192.0 10.112.164.2
no ip route 10.112.192.0 255.255.192.0 10.112.224.130
no ip route 10.113.0.0 255.255.128.0 10.113.66.2
no ip route 10.113.128.0 255.255.192.0 10.113.66.2
no ip route 10.113.128.0 255.255.224.0 192.168.14.2
no ip route 10.113.160.0 255.255.255.128 192.168.14.2
end
write memory
```
  
**Jangan menghapus default route R-PUSAT menuju ISP.**  
  
Default route tersebut tetap diperlukan agar R-PUSAT dapat mengarahkan trafik internet ke ISP dan mengumumkannya ke router lain melalui OSPF  
  
**B. Menghapus Static Route R-UTARA**  
  
```bash
enable
configure terminal
no ip route 0.0.0.0 0.0.0.0 10.113.66.1
no ip route 10.113.128.0 255.255.192.0 10.113.160.130
end
write memory
```
  
**C. Menghapus Static Route R-DC**  
  
```bash
enable
configure terminal
no ip route 0.0.0.0 0.0.0.0 10.112.97.1
end
write memory
```
  
**D. Menghapus Static Route R-BARAT**  
  
```bash
enable
configure terminal
no ip route 0.0.0.0 0.0.0.0 10.112.33.1
end
write memory
```
  
**E. Menghapus Static Route R-TIMUR**  
  
```bash
enable
configure terminal
no ip route 0.0.0.0 0.0.0.0 10.112.164.1
end
write memory
```
  
**F. Menghapus Static Route R-SELATAN**  
  
```bash
enable
configure terminal
no ip route 0.0.0.0 0.0.0.0 10.112.224.129
end
write memory
```
  
**G. Menghapus Static Route R-PELABUHAN**  
  
```bash
enable
configure terminal
no ip route 0.0.0.0 0.0.0.0 10.113.160.129
no ip route 10.113.192.0 255.255.248.0 192.168.14.1
no ip route 10.113.200.0 255.255.255.224 192.168.14.1
end
write memory
```

tambahkan ini juga

```
configure terminal
router ospf 1
 no passive-interface Tunnel0
 network 192.168.14.0 0.0.0.3 area 0
end
write memory
```
  
**Hasil yang Diharapkan**  
  
Setelah static route antar-router dihapus:
- R-PUSAT tetap memiliki default route menuju ISP.
- Router lain mendapatkan default route melalui OSPF.
- Jaringan antarwilayah dipelajari secara dinamis.
- Link cadangan dapat digunakan ketika jalur utama terputus.
  
**Langkah 5: Verifikasi Konfigurasi OSPF**  
  
Setelah konfigurasi selesai, tunggu proses pembentukan neighbor dan pertukaran informasi routing  
Proses ini dapat membutuhkan sekitar 30–60 detik tergantung kondisi topologi dan Packet Tracer  
  
**A. Memeriksa OSPF Neighbor**
  
Pada router, jalankan:
```bash
show ip ospf neighbor
```
  
Periksa kolom **State**  
Hubungan neighbor yang berhasil umumnya menunjukkan status:
```text
FULL
```
  
R-PUSAT diharapkan memiliki neighbor ke:
- R-UTARA
- R-DC
- R-BARAT
- R-TIMUR
- R-SELATAN
- R-PELABUHAN melalui Tunnel0
  
**B. Memeriksa Routing Table**
  
Jalankan:
```bash
show ip route
```
  
Troubleshoot  
Di show run | section router ospf R-PELABUHAN tidak ada baris no passive-interface Tunnel0. Tun0 sudah masuk OSPF (area 0, state POINT), tapi statusnya masih passive karena passive-interface default, jadi tidak mengirim hello dan Nbrs F/C tetap 0/0. R-BARAT sudah benar, tidak perlu diubah  

Di R-PELABUHAN
```
configure terminal
router ospf 1
 no passive-interface default
 passive-interface FastEthernet0/1
 passive-interface FastEthernet1/0
end
write memory
```

Di R-PUSAT
```
configure terminal
router ospf 1
 no passive-interface default
 passive-interface FastEthernet1/0
 passive-interface FastEthernet1/1
end
write memory
```
  
Di R-PUSAT & R-PELABUHAN  
```
configure terminal
interface Tunnel0
 bandwidth 6666
end
write memory
```
  
Naikkan cost L7
```
! R-DC
configure terminal
interface Ethernet0/1/0
 ip ospf cost 100
end
write memory

! R-UTARA
configure terminal
interface Ethernet0/0/0
 ip ospf cost 100
end
write memory
```
  
Cost link fisik dinaikkan, tunnel dibiarkan 1000. Aturannya:
- Jalur fisik ke LAN Pelabuhan harus lebih mahal dari 1000, supaya tunnel menang.
- Jalur fisik ke ujung tunnel (10.113.160.130 dan 10.113.66.1) harus tetap lebih murah dari tunnel, supaya tunnel tidak bergantung pada dirinya sendiri (recursive routing)  
  
Nilai yang memenuhi keduanya: L2 = 100, L6 = 950, L7 = 500  
  
```
! R-PUSAT
configure terminal
interface FastEthernet0/1
 ip ospf cost 100
end
write memory

! R-UTARA
configure terminal
interface FastEthernet0/0
 ip ospf cost 100
interface FastEthernet0/1
 ip ospf cost 950
interface Ethernet0/0/0
 ip ospf cost 500
end
write memory

! R-PELABUHAN
configure terminal
interface FastEthernet0/0
 ip ospf cost 950
end
write memory

! R-DC
configure terminal
interface Ethernet0/1/0
 ip ospf cost 500
end
write memory
```
  
Interface LAN (Fa0/1 dan Fa1/0 di Pelabuhan, Fa1/0 dan Fa1/1 di PUSAT) tetap passive, jadi syarat “tidak mengirim OSPF ke arah LAN” terpenuhi. Fa0/0 PUSAT (ke ISP) tidak ada di network statement, jadi tidak ikut OSPF  
  
Perhatikan kode routing berikut:
| Kode | Arti |
|---|---|
| `C` | Connected, jaringan terhubung langsung. |
| `L` | Local, alamat interface router. |
| `O` | Rute yang dipelajari melalui OSPF. |
| `O*E2` | Default route eksternal OSPF tipe 2. |
| `S` | Static route. |
| `S*` | Static default route. |
  
Router selain R-PUSAT diharapkan memiliki default route:
```text
O*E2 0.0.0.0/0
```
  
Artinya, default route telah dipelajari melalui OSPF  
Pada R-PUSAT, default route statis menuju ISP tetap dipertahankan  
  
**C. Memeriksa Passive Interface**  
  
Jalankan:
```bash
show ip protocols
```
  
Pastikan:
- Interface LAN menggunakan passive-interface.
- Interface antar-router tidak passive.
- Jaringan yang diumumkan sudah sesuai.
- OSPF berjalan pada Area 0.
  
Passive-interface digunakan agar router tidak mengirim paket Hello OSPF ke arah perangkat LAN, tetapi jaringan tersebut tetap dapat diumumkan melalui OSPF  
  
**D. Menguji Jalur Tunnel**
  
Dari **PC Pegawai Balai Kota**, buka:
  
**Desktop → Command Prompt**  
  
Jalankan perintah traceroute menuju IP PC Kantor Pelabuhan:
```bash
tracert <IP-PC-KANTOR-PELABUHAN>
```
  
Ganti `<IP-PC-KANTOR-PELABUHAN>` dengan alamat IP PC tujuan yang sebenarnya  
  
Jalur yang diharapkan melewati:
1. Gateway Balai Kota `10.113.192.1`.
2. Tunnel menuju Pelabuhan `192.168.14.2`.
3. Jaringan tujuan di Pelabuhan.
  
Jika muncul jalur melalui `10.113.66.2`, periksa kembali nilai OSPF cost atau status Tunnel0  
  
**E. Menguji Akses Internet dan Layanan**  
  
Setelah OSPF aktif, ulangi pengujian seluruh layanan  
  
**Tes Internet**
  
```bash
ping 8.8.8.8
```
  
**Tes DNS**
  
```bash
nslookup www.k14.smartcity.lab
```
   
**Tes Web**
  
Buka Web Browser lalu akses:
`http://www.k14.smartcity.lab`
  
**Tes FTP**
  
Dari PC CCTV:
  
```bash
ftp ftp.k14.smartcity.lab
```
  
Login dengan akun yang sudah dibuat, kemudian lakukan pengujian unggah file.
  
**Tes Email**  
  
Kirim email dari:
`pegawai@k14.smartcity.lab`
  
Ke:
`pelabuhan@k14.smartcity.lab`
  
Lalu lakukan pengujian sebaliknya  
Seluruh layanan diharapkan tetap berfungsi setelah static route digantikan oleh OSPF  
  
**Langkah 6: Pengujian Pemulihan Jaringan Saat Link Terputus**

Pengujian ini membuktikan bahwa OSPF memulihkan konektivitas secara otomatis lewat jalur lain ketika salah satu link backbone terputus. Link cadangan L7 (R-DC ke R-UTARA, `10.112.97.4/30`) menjadi jalur pengganti  
  
**Dasar desain**

Cost OSPF diatur supaya L7 hanya dipakai saat jalur utama putus  
  
| Link | Interface | Cost |
| :--- | :--- | :---: |
| L2 (R-PUSAT ke R-UTARA) | Fa0/1 R-PUSAT dan Fa0/0 R-UTARA | 100 |
| L6 (R-UTARA ke R-PELABUHAN) | Fa0/1 R-UTARA dan Fa0/0 R-PELABUHAN | 950 |
| L7 (R-DC ke R-UTARA) | Eth0/1/0 R-DC dan Eth0/0/0 R-UTARA | 500 |
| Tunnel GRE | Tunnel0 | 1000 (bawaan, tidak dapat diubah di Packet Tracer) |
  
Jalur normal dari Utara ke internet adalah R-UTARA, L2, lalu R-PUSAT. Jalur lewat L7 (R-UTARA, L7, R-DC, L1, R-PUSAT) bercost lebih besar sehingga hanya terpilih saat L2 atau L1 terputus  
  
**Skenario 1: Link L2 terputus (R-PUSAT ke R-UTARA)**
Tujuan: memastikan wilayah Utara tetap terhubung ke internet lewat L7  
  
Langkah pengujian:
1. Dari smartphone atau PC di LAN WiFi Utara, jalankan `ping 8.8.8.8 -t` dan `tracert 8.8.8.8` untuk kondisi normal.
2. Matikan L2 di R-PUSAT:
```
configure terminal
interface FastEthernet0/1
 shutdown
end
```
3. Tunggu sekitar 40 detik (dead timer OSPF), lalu jalankan `tracert 8.8.8.8` lagi dari perangkat yang sama.
4. Di R-PUSAT jalankan `show ip ospf neighbor` untuk melihat neighbor yang hilang.
5. Dari PC Balai Kota, jalankan `tracert` ke PC Kantor Pelabuhan.
  
Hasil yang diperoleh pada `tracert 8.8.8.8` dari Utara saat L2 mati:
| Hop | Alamat | Perangkat |
| :---: | :--- | :--- |
| 1 | `10.113.0.1` | R-UTARA |
| 2 | `10.112.97.5` | R-DC (sisi L7) |
| 3 | `10.112.97.1` | R-PUSAT (sisi L1) |
| 4 | `203.0.113.113` | R-ISP |
| 5 | `8.8.8.8` | Server internet |
  
Analisis:
- Trafik tidak lagi langsung ke R-PUSAT. Jalurnya memutar lewat L7 ke R-DC, lalu L1 ke R-PUSAT, kemudian keluar ke internet melalui NAT.
- Console R-PUSAT menampilkan Tunnel0 ikut turun dan neighbor `6.6.6.6` (R-UTARA) serta `7.7.7.7` (R-PELABUHAN lewat Tunnel0) hilang. Hal ini wajar karena sumber tunnel adalah Fa0/1 yang sedang dimatikan.
- Lalu lintas Balai Kota ke Pelabuhan berpindah ke jalur fisik (R-PUSAT, R-DC, R-UTARA, R-PELABUHAN), sehingga komunikasi tetap berjalan walaupun tunnel mati.

[Screenshot: tracert 8.8.8.8 dari Utara sebelum dan saat L2 mati]
[Screenshot: log console R-PUSAT saat L2 dimatikan]
[Screenshot: show ip ospf neighbor di R-PUSAT saat L2 mati]
[Screenshot: tracert Balai Kota ke Pelabuhan saat L2 mati]

Mengembalikan L2:

```
configure terminal
interface FastEthernet0/1
 no shutdown
end
```

Setelah sekitar 40 detik, `show ip ospf neighbor` di R-PUSAT kembali menampilkan enam neighbor berstatus FULL, termasuk `7.7.7.7` lewat Tunnel0. Traceroute Balai Kota ke Pelabuhan kembali melewati tunnel dengan tiga hop (`10.113.192.1`, `10.113.160.130`, lalu PC tujuan), tanpa hop R-UTARA.

[Screenshot: show ip ospf neighbor setelah L2 dihidupkan kembali]
[Screenshot: tracert Balai Kota ke Pelabuhan setelah pulih]

**Skenario 2: Link L1 terputus (R-PUSAT ke R-DC)**

Tujuan: memastikan wilayah DC (LAN CCTV Kota dan Server and Storage DC) tetap terhubung ke R-PUSAT dan internet lewat L7.

Langkah pengujian:

1. Dari PC CCTV Kota, jalankan `ping 8.8.8.8 -t` dan `tracert 8.8.8.8` untuk kondisi normal.
2. Matikan L1 di R-PUSAT:

```
configure terminal
interface Ethernet0/0/0
 shutdown
end
```

3. Tunggu sekitar 40 detik, lalu jalankan `tracert 8.8.8.8` dari PC CCTV.
4. Di R-PUSAT jalankan `show ip ospf neighbor` dan `show ip route`.
5. Dari PC Pegawai Balai Kota, uji akses ke Server DC dengan `ping 10.112.96.2`.

Hasil yang diharapkan pada `tracert 8.8.8.8` dari PC CCTV saat L1 mati:

| Hop | Alamat | Perangkat |
| :---: | :--- | :--- |
| 1 | `10.112.64.1` | R-DC (gateway LAN CCTV) |
| 2 | `10.112.97.6` | R-UTARA (sisi L7) |
| 3 | `10.113.66.1` | R-PUSAT (sisi L2) |
| 4 | `203.0.113.113` | R-ISP |
| 5 | `8.8.8.8` | Server internet |

Analisis:

- Karena L1 putus, R-DC tidak punya jalur langsung ke R-PUSAT. OSPF memindahkan jalurnya ke L7 menuju R-UTARA, lalu L2 menuju R-PUSAT.
- Neighbor `2.2.2.2` (R-DC) hilang dari tabel neighbor R-PUSAT, dan rute ke LAN DC di R-PUSAT kini mengarah ke `10.113.66.2` (R-UTARA).
- Tunnel GRE tidak terpengaruh karena L2 masih hidup, sehingga neighbor `7.7.7.7` tetap FULL.
- Server DC tetap dapat dijangkau dari wilayah lain, sehingga layanan DNS, FTP, dan email tetap berjalan.

[Screenshot: tracert 8.8.8.8 dari PC CCTV saat L1 mati]
[Screenshot: show ip ospf neighbor di R-PUSAT saat L1 mati]
[Screenshot: show ip route di R-PUSAT (rute ke LAN DC lewat 10.113.66.2)]
[Screenshot: ping dari PC Pegawai ke Server DC saat L1 mati]

Mengembalikan L1:

```
configure terminal
interface Ethernet0/0/0
 no shutdown
end
```

Setelah sekitar 40 detik, `show ip ospf neighbor` di R-PUSAT kembali menampilkan enam neighbor FULL dan `tracert` dari PC CCTV kembali lewat `10.112.64.1`, lalu `10.112.97.1`, lalu `203.0.113.113`.

[Screenshot: show ip ospf neighbor setelah L1 dihidupkan kembali]

**Catatan pengujian**

- Link L6 (R-UTARA ke R-PELABUHAN) tidak diuji karena R-PELABUHAN hanya memiliki satu link ke bagian atas jaringan. Kondisi ini adalah keterbatasan topologi pada Gambar 1, bukan kesalahan konfigurasi.
- Waktu pemulihan sekitar 40 detik sesuai dead interval OSPF (4 kali hello interval 10 detik).
- Setelah pengujian, semua interface dikembalikan ke kondisi aktif dan diverifikasi dengan `show ip interface brief` (tidak ada status `administratively down`).

**Kesimpulan**

Ketika L2 maupun L1 terputus, OSPF otomatis menghitung ulang rute dan memakai L7 sebagai jalur cadangan tanpa konfigurasi manual. Setelah link dipulihkan, jalur kembali ke rute utama sesuai cost yang telah ditetapkan.
**Kesimpulan Poin 7: Konfigurasi Routing OSPF**

Konfigurasi OSPF dinyatakan berhasil apabila:

- Seluruh router internal menjalankan OSPF Area 0.
- Router membentuk hubungan neighbor dengan status FULL.
- Jaringan LAN dapat dipelajari melalui OSPF.
- Interface LAN tidak mengirim paket Hello OSPF karena menggunakan passive-interface.
- Router selain R-PUSAT memperoleh default route melalui OSPF.
- R-PUSAT tetap menggunakan default route statis menuju ISP.
- Trafik Balai Kota menuju Pelabuhan menggunakan tunnel selama jalur tersedia.
- Link L7 dapat digunakan sebagai jalur cadangan ketika link utama mengalami gangguan.
- Layanan DNS, Web, FTP, dan Email tetap dapat diakses setelah penerapan OSPF.

**Penyimpanan Akhir**

Setelah semua pengujian berhasil:

1. Jalankan `write memory` pada semua router.
2. Simpan file menggunakan **File → Save**.
3. Pastikan file yang digunakan adalah:

   `K14_CIDR_OSPF.pkt`

4. Pertahankan file asli:

   `K14_CIDR.pkt`

Kedua file tersebut digunakan untuk membedakan konfigurasi jaringan sebelum dan sesudah penerapan OSPF.

---

**Tahap Selanjutnya: Dokumentasi GitHub dan Spreadsheet**

Setelah konfigurasi teknis selesai, tahap berikutnya adalah merapikan dokumentasi proyek.

Dokumentasi dapat mencakup:

- Topologi jaringan dan pembagian subnet menggunakan CIDR.
- Tabel IP address, subnet mask, dan default gateway.
- Konfigurasi layanan DNS, Web, FTP, dan Email.
- Konfigurasi routing OSPF.
- Hasil pengujian konektivitas antarwilayah.
- Hasil pengujian failover menggunakan link L7.
- Screenshot konfigurasi dan pengujian.
- File Packet Tracer sebelum dan sesudah penerapan OSPF.

Seluruh dokumentasi dapat disusun menggunakan Markdown untuk GitHub dan spreadsheet untuk tabel pembagian alamat IP.


Login dan pindahkan file yang telah dibuat di text editor
Perintah dir menampilkan file yang telah dipindahkan