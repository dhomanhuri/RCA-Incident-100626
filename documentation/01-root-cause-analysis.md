# Root Cause Analysis — Traffic Drop MGW2-CYB1
**Tanggal incident:** 2026-06-10  
**Traffic drop:** ~20:20 WIB  
**Recovery:** ~21:20 WIB  
**Durasi:** ±1 jam  

---

## 1. Informasi Device

### MikroTik (MGW2-CYB1)
| Parameter | Detail |
|---|---|
| Hostname | MGW2-CYB1 |
| IP | 150.242.176.185 |
| Model | CCR2116-12G-4S+ |
| RouterOS | 7.18.2 (stable) |
| Uptime | 6 minggu 1 hari (tidak ada reboot saat incident) |

### Juniper (IGW1-CYB1) — Upstream Router
| Parameter | Detail |
|---|---|
| Hostname | IGW1-CYB1 |
| IP | 150.242.176.161 |
| Model | MX240 |
| JunOS | 21.4R3.15 |
| Uptime | 775 hari (tidak ada reboot saat incident) |

---

## 2. Topologi

```
Internet (IIX/Transit)
    │
    ├── BGP Peer 113.59.234.208 (AS 45296)  — International Transit
    ├── BGP Peer 123.108.8.111  (AS 7597)   — Domestic (IIX) ⚠️
    ├── BGP Peer 123.108.9.111  (AS 7597)   — Domestic (IIX)
    ├── BGP Peer 10.9.32.1      (AS 150930) — Direct Peering IPTV
    └── BGP Peer 150.242.176.182 (AS 152069) — Internal
         │
         ▼
    IGW1-CYB1 (Juniper MX240) — 150.242.176.161
         │
         ▼ xe-2/0/2.11 (150.242.176.161/27)
    MGW2-CYB1 (MikroTik CCR2116) — 150.242.176.185
         │
         ├── DOWNLINK sfp-sfpplus1
         └── UPLINK sfp-sfpplus2
              ├── v515_STARLINK_BULK1 (primary)
              └── v531_STARLINK_BULK2 (secondary)
```

---

## 3. Temuan Investigasi

### 3.1 Log MikroTik — Tidak Tersedia ❌
Log jam 20:00–21:30 WIB sudah **tertimpa** oleh `radius,debug` logging yang sangat verbose. Buffer log habis.

### 3.2 Log Juniper — Tidak Dapat Diakses ❌
User `plm1` tidak memiliki permission untuk membaca `/var/log/messages` di Juniper (`error: permission denied: log`).

### 3.3 Interface Fisik MikroTik — Normal ✅
Tidak ada RX error atau TX drop signifikan pada sfp-sfpplus1 dan sfp-sfpplus2.

### 3.4 Interface xe-2/0/2 Juniper (Link ke MikroTik) — Ada Error ⚠️

| Metrik | Nilai |
|---|---|
| Last flapped | 2024-09-19 (90 minggu lalu — normal, tidak terkait incident) |
| Input errors | 26,766 (L3 incompletes: 26,758) |
| **MTU errors (output)** | **5,677,146** |
| Carrier transitions | 5 |
| PCS Bit errors | 5 seconds |

**MTU errors sebanyak 5,677,146** adalah temuan signifikan — menunjukkan ada ketidakcocokan MTU antara Juniper dan perangkat di downstream (MikroTik atau Starlink).

### 3.5 BGP di Juniper — Temuan Kritis ⚠️

| BGP Peer | AS | Flaps | Last Up | Tipe | Keterangan |
|---|---|---|---|---|---|
| 10.9.32.1 | 150930 | 53 | 6w5d | Direct Peering (IPTV) | Normal |
| 113.59.234.208 | 45296 | 75 | 6w6d | International Transit | Normal |
| **123.108.8.111** | **7597** | **525** | **5d 10:04** | **Domestic (IIX)** | **KRITIS ⚠️** |
| 123.108.9.111 | 7597 | 117 | 4w4d | Domestic (IIX) | Elevated |
| 150.242.176.182 | 152069 | 109 | 3w0d | Internal | Elevated |

**BGP Peer 123.108.8.111 (AS 7597 / Domestic IIX):**
- Flaps: **525** — sangat tinggi
- Last up: **hanya 5 hari** — artinya sering disconnect-reconnect
- Last error: `Hold Timer Expired Error`
- Error detail: Hold Timer Expired dikirim 75x, Open Message Error dikirim 163x
- Last flap event: `HoldTime`

Ini menunjukkan **BGP peer IIX mengalami instabilitas yang berulang**, termasuk kemungkinan besar di sekitar jam 20:20 WIB.

---

## 4. Root Cause — Probable

> **BGP Instability pada peer Domestic IIX (123.108.8.111 / AS 7597)**

### Mekanisme

```
BGP peer 123.108.8.111 (Domestic IIX) Hold Timer Expired
    │
    ▼
BGP session drop → route withdrawal (domestic routes)
    │
    ▼
IGW1-CYB1 kehilangan routing table domestic dari IIX
    │
    ▼
Traffic ke domestic prefix tidak ada route aktif
(International transit & IPTV direct peering tetap aktif)
    │
    ▼
Traffic drop ~6-8% di MGW2-CYB1 (domestic traffic terdampak)
    │
    ▼
~60 menit kemudian BGP re-establish → route kembali
    │
    ▼
Traffic recovery ~21:20 WIB
```

### Faktor Pendukung
1. **525 flaps** pada peer IIX — instabilitas kronis, bukan satu kejadian
2. **Last up hanya 5 hari** — peer sering disconnect
3. **Hold Timer Expired** — koneksi BGP timeout, bukan pemutusan disengaja
4. **MTU errors 5.7 juta** — kemungkinan berkontribusi pada packet loss yang memperburuk BGP keepalive

---

## 5. Rekomendasi

### Segera

| # | Tindakan | Target |
|---|---|---|
| 1 | Investigasi stabilitas BGP peer 123.108.8.111 (IIX) | Tim Network / NOC |
| 2 | Konfirmasi ke IIX apakah ada gangguan jam 20:20 WIB 10 Juni | Tim Peering |
| 3 | Nonaktifkan `radius,debug` logging di MikroTik | MGW2-CYB1 |
| 4 | Investigasi MTU mismatch di interface xe-2/0/2 Juniper | Tim Network |

### Perintah MikroTik — Nonaktifkan Radius Debug
```bash
/system logging disable [find topics~"radius,debug"]
```

### Jangka Panjang

| # | Tindakan |
|---|---|
| 5 | Aktifkan syslog ke server eksternal di MikroTik |
| 6 | Tambahkan monitoring BGP flap (alert jika peer down) |
| 7 | Pasang Grafana/SNMP monitoring untuk traffic historis |
| 8 | Review konfigurasi BGP holdtime dan keepalive timer |
| 9 | Investigasi dan perbaiki MTU errors di xe-2/0/2 |
| 10 | Pertimbangkan BGP route dampening untuk peer yang flappy |

---

## 5.5 Konfirmasi Traffic Drop dari Observium NMS ✅

Data grafik traffic MGW2-CYB1 dari Observium NMS (`https://nms.nsc.id`) mengkonfirmasi incident:

| Waktu (WIB) | Traffic Level | Status |
|---|---|---|
| 19:00 – 20:18 | 98–100% | Normal |
| **20:23 – 21:12** | **92–94%** | **DROP ~6-8%** |
| 21:17 dst | 99–100% | Recovery |

**Drop dimulai tepat ~20:23 WIB dan recovery ~21:17 WIB** — sesuai laporan incident.

Drop sebesar ~6-8% dari total traffic device mengindikasikan sebagian traffic (bukan semua) terdampak — **konsisten dengan BGP peer partial failure** dimana hanya route yang diiklankan peer IIX yang hilang, sementara route transit tetap aktif.

---

## 6. Status Investigasi

| Item | Status |
|---|---|
| Investigasi MikroTik | ✅ Selesai |
| Investigasi Juniper IGW1-CYB1 | ✅ Selesai (partial — log tidak bisa diakses) |
| Identifikasi BGP peer bermasalah | ✅ Ditemukan (123.108.8.111 / IIX) |
| Konfirmasi ke IIX | ⏳ Pending |
| Fix radius debug logging | ⏳ Pending |
| Fix MTU mismatch | ⏳ Pending |

---

*Investigasi dilakukan: 2026-06-10 23:36 WIB & 2026-06-11 10:00 WIB*  
*Investigator: Ilyasai (AI Assistant)*  
*Confidence: **Medium-High** — BGP peer IIX adalah kandidat root cause paling kuat, perlu konfirmasi dari IIX*
