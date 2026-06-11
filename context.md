# Context — routecauseanalisysmikrotik

## Tujuan Project
Root cause analysis traffic drop pada MikroTik MGW2-CYB1 yang terjadi pada 2026-06-10 jam 20:20 WIB dan recovery 21:20 WIB (durasi ±1 jam).

## Informasi Device
- **Hostname:** MGW2-CYB1
- **IP:** 150.242.176.185
- **Model:** CCR2116-12G-4S+
- **OS:** RouterOS 7.18.2 (stable)
- **Uptime saat investigasi:** 6w1d (tidak ada reboot)

## Incident
| Key | Detail |
|---|---|
| Tanggal | 2026-06-10 |
| Traffic drop | ~20:20 WIB |
| Traffic recovery | ~21:20 WIB |
| Durasi | ±1 jam |

## Topologi Relevan
- UPLINK: sfp-sfpplus2
- DOWNLINK: sfp-sfpplus1
- Dua koneksi Starlink sebagai redundansi:
  - v515_STARLINK_BULK1 (primary, traffic sangat besar ~29.8 TB)
  - v531_STARLINK_BULK2 (secondary, traffic kecil ~1.1 GB)
- VRRP untuk failover antar Starlink:
  - vrrp_Prefer_Mikrotik1: interface v531_STARLINK_BULK2, priority 204 → saat ini BACKUP
  - vrrp_Prefer_Mikrotik2: interface v515_STARLINK_BULK1, priority 99 → saat ini MASTER
- Default route: 150.242.176.161 (static, distance 1)

## Temuan Investigasi
1. Log jam 20:00-21:30 WIB sudah tertimpa oleh radius debug yang sangat verbose
2. VRRP kondisi tidak normal:
   - vrrp_Prefer_Mikrotik1 (priority 204) → BACKUP (seharusnya MASTER)
   - vrrp_Prefer_Mikrotik2 (priority 99) → MASTER
   - IP 10.20.2.3/24 (milik Mikrotik1) → DISABLED
   - IP 10.20.2.2/24 (milik Mikrotik2) → ACTIVE
3. Starlink Bulk2 (v531) traffic sangat kecil dibanding Bulk1 → menunjukkan kondisi yang tidak normal
4. Interface fisik sfp-sfpplus1 dan sfp-sfpplus2 tidak ada error (rx-fcs-error=0, rx-overflow=0)

## Hipotesis Root Cause
VRRP failover — Mikrotik1 kehilangan peran MASTER dan berpindah ke BACKUP karena kemungkinan:
- Starlink Bulk2 mengalami gangguan (link drop/flap) sehingga VRRP failover terpicu
- Selama ±1 jam traffic dialihkan melalui jalur backup hingga recovery

## Temuan Final
- Log MikroTik tertimpa radius debug → tidak bisa direcovery
- Log Juniper tidak bisa diakses (permission denied)
- Interface fisik MikroTik normal
- VRRP kondisi normal sesuai desain
- **BGP peer IIX (123.108.8.111/AS7597) sangat flappy: 525 flaps, last up hanya 5 hari**
- MTU errors 5.7 juta di interface Juniper xe-2/0/2 (link ke MikroTik)
- **Root cause probable: BGP instability pada peering IIX**

## Status
- [x] Investigasi MikroTik selesai
- [x] Investigasi Juniper IGW1-CYB1 selesai (partial)
- [x] Root cause probable teridentifikasi
- [ ] Konfirmasi ke IIX (123.108.8.111/AS7597) jam incident
- [ ] Nonaktifkan radius debug logging di MikroTik
- [ ] Setup syslog eksternal
- [ ] Fix MTU mismatch di xe-2/0/2

## Log Permintaan
- [2026-06-10] Inisialisasi investigasi, SSH ke MikroTik berhasil
- [2026-06-10] Investigasi interface, log, VRRP, routing, dan queue MikroTik
- [2026-06-10] Koreksi: kondisi VRRP normal by design
- [2026-06-11] SSH ke Juniper IGW1-CYB1 (150.242.176.161) berhasil
- [2026-06-11] Temuan kritis: BGP peer IIX 123.108.8.111 flaps 525x, last up 5 hari
- [2026-06-11] Dokumentasi final diupdate dengan root cause probable
- [2026-06-11] Akses Observium NMS (nms.nsc.id) berhasil
- [2026-06-11] Traffic drop terkonfirmasi dari grafik: drop 20:23-21:17 WIB, sebesar ~6-8%
- [2026-06-11] Drop partial (bukan total) → konsisten dengan BGP peer partial failure (IIX)
