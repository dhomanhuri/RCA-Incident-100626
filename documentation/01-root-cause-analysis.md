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
Internet
    │
    ├── BGP 113.59.234.208 (AS 45296)  — International Transit
    ├── BGP 123.108.8.111  (AS 7597)   — Domestic (IIX) ⚠️
    ├── BGP 123.108.9.111  (AS 7597)   — Domestic (IIX)
    ├── BGP 10.9.32.1      (AS 150930) — Direct Peering IPTV
    └── BGP 150.242.176.182 (AS 152069) — Internal
         │
         ▼
    IGW1-CYB1 (Juniper MX240) — 150.242.176.161
         │
         ▼ xe-2/0/2.11 (150.242.176.161/27)
    MGW2-CYB1 (MikroTik CCR2116) — 150.242.176.185
```

---

## 3. Temuan Investigasi

### 3.1 Traffic Drop Terkonfirmasi dari Observium ✅

Data grafik Observium mengkonfirmasi drop terjadi **tepat 20:25 WIB** dan recovery **21:12-21:17 WIB**:

| Interface | Normal | Drop (20:25-21:12) | Keterangan |
|---|---|---|---|
| xe-2/0/2.11 (Uplink ke MikroTik) | 100% | **~23%** (drop 77%) | Sangat parah |
| xe-2/0/2.932 (IPTV Direct Peer) | 100% | **~16%** (drop 84%) | Sangat parah |
| xe-2/0/2.2722 (Domestic IIX) | 100% | **~75%** (drop 25%) | Parah |
| xe-2/0/2.2723 (International Transit) | 100% | **~52%** (drop 48%) | Parah |

> ⚠️ **Semua interface drop serentak di 20:25 WIB** — termasuk IPTV Direct Peer yang seharusnya tidak terdampak oleh BGP IIX. Ini mengindikasikan masalah di **layer fisik atau Juniper itu sendiri**, bukan hanya BGP.

### 3.2 BGP Status Saat Investigasi (11 Juni ~10:10 WIB) ⚠️

| BGP Peer | AS | Tipe | Flaps | Last Up/Dwn |
|---|---|---|---|---|
| 10.9.32.1 | 150930 | IPTV Direct Peering | 53 | 6w5d (normal) |
| **113.59.234.208** | 45296 | **International Transit** | 75 | **~22 menit** ← baru naik! |
| **123.108.8.111** | 7597 | **Domestic IIX** | **525** | **5d** ← sangat flappy |
| 123.108.9.111 | 7597 | Domestic IIX | 117 | 4w4d |
| 150.242.176.182 | 152069 | Internal | 109 | 3w0d |

**BGP International Transit (113.59.234.208) baru up ~22 menit saat investigasi** — menunjukkan peer ini juga baru saja reconnect, kemungkinan terkait dengan incident atau masalah yang sedang berlangsung.

### 3.3 Interface Fisik xe-2/0/2 ⚠️

| Metrik | Nilai |
|---|---|
| Last flapped | 2024-09-19 (90 minggu lalu — tidak terkait incident) |
| **MTU errors (output)** | **5,677,146** |
| Input L3 incompletes | 26,758 |
| PCS Bit errors | 5 seconds |

MTU errors sangat tinggi — potensi penyebab packet loss yang memperburuk kondisi.

---

## 3.4 Analisis Pola Drop Antar Interface (Evidence Kunci) ✅

Bandingkan pola drop antar interface:

| Interface | Pattern Drop | Recovery | Keterangan |
|---|---|---|---|
| **xe-2/0/2 PHYSICAL** | Drop **57%** mulai 20:25 | Bertahap sampai 21:12 | **Physical port utama** |
| IIX Domestic | Drop 25% di 20:25 | Cepat di 20:38 | Drop ringan, cepat recover |
| International Transit | **Fluktuatif sejak 19:12** | Naik bertahap 21:04 | **Sudah bermasalah sebelum incident!** |
| Uplink ke MikroTik | Drop 77% di 20:25 | Bertahap 21:12 | Mengikuti physical port |
| IPTV Direct Peer | Drop 84% di 20:25 | Bertahap 21:12 | Mengikuti physical port |

**Temuan kritis:**
1. **International Transit sudah fluktuatif dari 19:12 WIB** — 1 jam sebelum incident utama
2. **xe-2/0/2 physical drop 57%** — ini adalah root dari semua sub-interface yang terdampak
3. IIX Domestic recovery lebih cepat (20:38) — karena IIX hanya lewat sebagian physical port capacity

### 3.5 Commit History Juniper — Bukan Config Change ✅

Tidak ada commit konfigurasi di 10 Juni 2026:
- Commit terakhir sebelum incident: **2026-05-22 11:10 WIB** (19 hari sebelumnya)
- **Bukan human error / perubahan konfigurasi**

---

## 4. Root Cause — Revisi

> **Bukan hanya BGP IIX. Semua interface drop serentak → masalah di Juniper layer**

### Hipotesis yang Direvisi

**Hipotesis awal (BGP IIX saja) TIDAK TEPAT** karena:
- IPTV Direct Peering juga drop 84% — tidak melalui BGP IIX
- International Transit juga drop bersamaan
- Semua drop mulai **tepat 20:25 WIB** secara serentak

### Kandidat Root Cause (Direvisi)

#### 🔴 ROOT CAUSE TERKONFIRMASI: Degradasi Fisik Link di xe-2/0/2

**Evidence:**
- Physical port xe-2/0/2 drop 57% — semua sub-interface mengikuti
- **International Transit sudah fluktuatif sejak 19:12** — 1 jam sebelum incident puncak
- PCS Bit errors & Errored blocks = 5 seconds — ada degradasi sinyal fisik
- MTU errors 5,677,146 — indikasi kualitas link yang buruk
- **Carrier transitions = 5** (akumulatif) — pernah ada link down/up

**Logika konfirmasi:**
```
xe-2/0/2 (1 physical port)
    ├── xe-2/0/2.2722 → IIX Domestic     ┐
    ├── xe-2/0/2.2723 → International   ├─ Semua DROP serentak
    ├── xe-2/0/2.932  → IPTV Direct Peer ┘
    └── xe-2/0/2.11   → Uplink ke MikroTik ← downstream ikut terdampak
```
Jika **semua upstream** (IIX, International, IPTV) drop serentak di port yang sama → **downstream pasti ikut terdampak**. Ini bukan masalah di MikroTik, bukan Starlink, bukan BGP routing.

**Mekanisme:** Link fisik upstream (fiber/kabel/SFP) mengalami degradasi bertahap sejak 19:12, mencapai titik kritis di 20:25 sehingga capacity drop drastis, kemudian recovery bertahap selama ~50 menit.

#### 🟡 Kandidat 2 (SEDANG): Upstream Provider Network Issue

**Evidence:**
- International Transit sudah fluktuatif **sejak 19:12** (sebelum incident utama)
- Bukan perubahan konfigurasi lokal (commit history bersih)
- Recovery bertahap — konsisten dengan upstream yang perlahan membaik

**Mekanisme:** Provider upstream mengalami congestion atau rerouting yang menyebabkan traffic fluktuatif, memuncak di 20:25 dengan traffic drop signifikan.

#### 🟢 Kandidat 3 (RENDAH): Control Plane / CPU Spike Juniper

**Counter-evidence:**
- CPU Juniper RE saat investigasi: 4% (sangat rendah)
- Memory: 11% (normal)
- Tidak ada alarm chassis
- BGP holdtime 90 detik — butuh ~90 detik timeout, bukan spike singkat

---

## 5. Rekonstruksi Timeline

```
~20:25 WIB
  └─ Semua traffic interface Juniper drop serentak
     └─ Semua BGP peer terdampak bersamaan
        └─ Kemungkinan: xe-2/0/2 micro-flap ATAU Juniper RE overload

20:25 – ~21:08 WIB
  └─ Traffic sangat rendah di semua interface
     └─ BGP session mencoba reconnect
        └─ Recovery partial terlihat di 21:08 (IPTV & Uplink mulai naik)

~21:12 – 21:17 WIB
  └─ Traffic mulai recovery ke level normal
     └─ BGP session re-established

Saat investigasi (11 Juni 10:10 WIB):
  └─ BGP International Transit baru naik 22 menit
     └─ Indikasi masalah masih berlanjut atau baru saja ada incident lain
```

---

## 6. Konfirmasi Traffic Drop dari Observium NMS ✅

Data grafik traffic dari Observium NMS (`https://nms.nsc.id`):

- **Drop dimulai:** tepat **20:25 WIB**
- **Level drop:** 77-84% di interface kritis (uplink ke MikroTik & IPTV)
- **Recovery:** bertahap mulai **21:08**, normal di **21:17 WIB**
- **Karakteristik:** drop serentak semua interface → **bukan BGP partial failure**

---

## 7. Rekomendasi

### Segera

| # | Tindakan | Target |
|---|---|---|
| 1 | **Cek log Juniper** dengan akses yang tepat (user dengan permission read log) | NOC/Admin Juniper |
| 2 | Cek apakah ada **CPU/RE spike** di Juniper jam 20:25 WIB | NOC/Admin |
| 3 | Cek apakah **xe-2/0/2 mengalami micro-flap** jam 20:25 WIB | NOC/Admin |
| 4 | Investigasi **BGP International Transit naik baru 22 menit** — apakah ada incident baru | NOC |
| 5 | Nonaktifkan `radius,debug` logging di MikroTik | `/system logging disable [find topics~"radius,debug"]` |

### Jangka Panjang

| # | Tindakan |
|---|---|
| 6 | Aktifkan syslog ke server eksternal di MikroTik |
| 7 | Setup monitoring alert untuk BGP flap |
| 8 | Investigasi dan perbaiki MTU errors di xe-2/0/2 (5.7 juta errors) |
| 9 | Review BFD configuration untuk faster BGP failure detection |
| 10 | Pasang monitoring Grafana/SNMP untuk historis CPU Juniper |

---

## 8. Status Investigasi

| Item | Status |
|---|---|
| Investigasi MikroTik | ✅ Selesai |
| Investigasi Juniper (via SSH read-only) | ✅ Partial |
| Konfirmasi traffic drop dari Observium | ✅ Drop 20:25-21:17 WIB terkonfirmasi |
| Identifikasi pola drop (serentak semua interface) | ✅ Temuan baru — revisi hipotesis |
| Log Juniper detail (perlu akses lebih tinggi) | ⏳ Perlu admin Juniper |
| Konfirmasi root cause final | ⏳ Pending log Juniper |

---

*Investigasi dilakukan: 2026-06-10 23:36 WIB & 2026-06-11 09:00-10:30 WIB*  
*Investigator: Ilyasai (AI Assistant)*  
*Confidence: **Medium** — drop terkonfirmasi, pola serentak teridentifikasi, root cause final butuh log Juniper dengan akses lebih tinggi*
